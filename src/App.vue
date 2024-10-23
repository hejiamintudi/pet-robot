<!-- ||src/components/ConnectionStatus.vue|| -->
<template>
  <div>
    <div class="device-actions">
      <button @click="startStreaming()"> 直播 </button>
      <button @click="startViewing()"> 观看 </button>
    </div>
    <video id="localVideoid" v-if="isShowLocalVideo" ref="localVideo" autoplay playsinline muted></video>
    <video id="remoteVideoid" v-if="isShowRemoteVideo" ref="remoteVideo" autoplay playsinline></video>
  </div>
</template>

<script>
import { ref } from 'vue';
import { SignalingClient, Role } from 'amazon-kinesis-video-streams-webrtc';
import AWS from 'aws-sdk';
import { config } from '@/config';
export default {
  name: 'PetRobot',
  setup() {
    let isShowLocalVideo = ref(false);
    let isShowRemoteVideo = ref(false);
    return {
      region: config.region, // 替换为您的区域
      accessKeyId: config.accessKeyId,
      secretAccessKey: config.secretAccessKey,
      channelName: config.channelName, // 替换为您的信令通道名称

      masterSignalingClient: null,
      masterPeerConnection: null,
      localStream: null,
      viewerSignalingClient: null,
      viewerPeerConnection: null,
      remoteStream: null,

      isShowLocalVideo,
      isShowRemoteVideo,

      islog: true,
    };
  },
  methods: {
    showLog() {

    },
    async getChannelARN(role) {
      const kinesisVideoClient = new AWS.KinesisVideo({
        region: this.region,
        credentials: new AWS.Credentials({
          accessKeyId: this.accessKeyId,
          secretAccessKey: this.secretAccessKey,
        }),
        correctClockSkew: true,
      });

      try {
        const describeSignalingChannelResponse = await kinesisVideoClient
          .describeSignalingChannel({
            ChannelName: this.channelName,
          })
          .promise();

        return describeSignalingChannelResponse.ChannelInfo.ChannelARN;
      } catch (error) {
        if (error.code === 'ResourceNotFoundException' && role === Role.MASTER) {
          console.log('信令通道不存在，正在创建...');
          const createSignalingChannelResponse = await kinesisVideoClient
            .createSignalingChannel({
              ChannelName: this.channelName,
              ChannelType: 'SINGLE_MASTER',
              SingleMasterConfiguration: {
                MessageTtlSeconds: 60,
              },
            })
            .promise();
          return createSignalingChannelResponse.ChannelARN;
        } else {
          throw error;
        }
      }
    },
    async startStreaming() {
      try {
        console.log("开始直播...");
        this.isShowLocalVideo = true;
        this.isShowRemoteVideo = false;

        if (!this.channelName) {
          alert('请填写信令通道名称。');
          return;
        }

        const channelARN = await this.getChannelARN(Role.MASTER);
        const clientId = null; // Master 不需要 clientId

        // 配置 AWS
        AWS.config.update({
          region: this.region,
          credentials: new AWS.Credentials({
            accessKeyId: this.accessKeyId,
            secretAccessKey: this.secretAccessKey,
          }),
          correctClockSkew: true,
        });

        // 创建 KinesisVideoClient
        const kinesisVideoClient = new AWS.KinesisVideo();

        // 获取信令通道终端节点
        const getSignalingChannelEndpointResponse = await kinesisVideoClient
          .getSignalingChannelEndpoint({
            ChannelARN: channelARN,
            SingleMasterChannelEndpointConfiguration: {
              Protocols: ['HTTPS', 'WSS'],
              Role: Role.MASTER,
            },
          })
          .promise();

        const endpointsByProtocol = getSignalingChannelEndpointResponse.ResourceEndpointList.reduce(
          (endpoints, endpoint) => {
            endpoints[endpoint.Protocol] = endpoint.ResourceEndpoint;
            return endpoints;
          },
          {}
        );

        // 创建信令客户端
        this.masterSignalingClient = new SignalingClient({
          channelARN,
          channelEndpoint: endpointsByProtocol.WSS,
          role: Role.MASTER,
          region: this.region,
          credentials: AWS.config.credentials,
          systemClockOffset: kinesisVideoClient.config.systemClockOffset,
          clientId,
        });

        // 获取 ICE 服务器配置
        const kinesisVideoSignalingChannelsClient = new AWS.KinesisVideoSignalingChannels({
          region: this.region,
          endpoint: endpointsByProtocol.HTTPS,
          credentials: AWS.config.credentials,
          correctClockSkew: true,
        });

        const getIceServerConfigResponse = await kinesisVideoSignalingChannelsClient
          .getIceServerConfig({
            ChannelARN: channelARN,
          })
          .promise();

        const iceServers = [
          { urls: `stun:stun.kinesisvideo.${this.region}.amazonaws.com:443` },
          ...getIceServerConfigResponse.IceServerList.map((iceServer) => ({
            urls: iceServer.Uris,
            username: iceServer.Username,
            credential: iceServer.Password,
          })),
          // TODO: 添加外部TURN服务器（仅用于测试）
          {
            urls: 'turn:27.25.138.55:3478',
            username: 'demo',
            credential: 'HSN95pGqr3QECqWe',
          }
        ];

        console.log('ICE Servers:', iceServers);

        // 创建 PeerConnection
        // TODO: iceTransportPolicy: 'relay'
        this.masterPeerConnection = new RTCPeerConnection({ iceServers });

        // 监控 ICE 连接状态
        this.masterPeerConnection.addEventListener('iceconnectionstatechange', () => {
          console.log('Master: ICE连接状态变化为:', this.masterPeerConnection.iceConnectionState);
        });

        // 处理 ICE 候选并添加日志
        this.masterPeerConnection.addEventListener('icecandidate', (event) => {
          if (event.candidate) {
            console.log('Master: 收到 ICE Candidate:', event.candidate.candidate);
            this.masterSignalingClient.sendIceCandidate(event.candidate);
          } else {
            console.log('Master: 所有ICE候选已收集完毕');
          }
        });

        // 添加本地流
        try {
          // 列出所有可用的媒体设备
          const devices = await navigator.mediaDevices.enumerateDevices();
          const videoDevices = devices.filter(device => device.kind === 'videoinput');

          if (videoDevices.length === 0) {
            console.log('没有找到视频输入设备');
            return;
          }

          // 使用第一个找到的视频设备
          this.localStream = await navigator.mediaDevices.getUserMedia({
            video: { deviceId: { exact: videoDevices[0].deviceId } },
            audio: false // 如果需要音频，可以设置为 true 或配置具体的音频设备
          });

          this.localStream.getTracks().forEach((track) => {
            this.masterPeerConnection.addTrack(track, this.localStream);
          });
          this.$refs.localVideo.srcObject = this.localStream;
          console.log('Master: 成功获取本地媒体流');
        } catch (error) {
          console.log('Master: 获取本地媒体流失败：' + error.message);
          return;
        }

        // 处理信令客户端事件
        this.masterSignalingClient.on('open', () => {
          console.log('Master: 信令客户端已连接');
        });

        this.masterSignalingClient.on('sdpOffer', async (offer, remoteClientId) => {
          try {
            console.log('Master: 收到 SDP Offer');
            await this.masterPeerConnection.setRemoteDescription(offer);

            // 创建 SDP Answer
            const answer = await this.masterPeerConnection.createAnswer();
            await this.masterPeerConnection.setLocalDescription(answer);

            // 发送 SDP Answer
            this.masterSignalingClient.sendSdpAnswer(this.masterPeerConnection.localDescription, remoteClientId);
            console.log('Master: 发送 SDP Answer');
          } catch (error) {
            console.error('Master: 处理 SDP Offer 失败', error);
          }
        });

        this.masterSignalingClient.on('iceCandidate', async (candidate, remoteClientId) => {
          try {
            console.log('Master: 收到 ICE 候选:', candidate.candidate);
            await this.masterPeerConnection.addIceCandidate(candidate);
          } catch (error) {
            console.error('Master: 添加ICE候选失败', error);
          }
        });

        this.masterSignalingClient.on('close', () => {
          console.log('Master: 信令客户端已关闭');
        });

        this.masterSignalingClient.on('error', (error) => {
          console.log('Master: 信令客户端错误：' + error.message);
        });

        // 开启信令客户端
        this.masterSignalingClient.open();
        console.log('Master: 信令客户端正在连接...');
      } catch (error) {
        console.log('Master错误：' + error.message);
      }
    },
    async startViewing() {
      try {
        console.log('开始观看直播...');
        this.isShowLocalVideo = false;
        this.isShowRemoteVideo = true;

        // 验证输入
        if (!this.channelName) {
          alert('请填写信令通道名称。');
          return;
        }

        const channelARN = await this.getChannelARN(Role.VIEWER);
        const clientId = 'viewer-' + Date.now();

        // 配置 AWS
        AWS.config.update({
          region: this.region,
          credentials: new AWS.Credentials({
            accessKeyId: this.accessKeyId,
            secretAccessKey: this.secretAccessKey,
          }),
          correctClockSkew: true,
        });

        // 创建 KinesisVideoClient
        const kinesisVideoClient = new AWS.KinesisVideo();

        // 获取信令通道终端节点
        const getSignalingChannelEndpointResponse = await kinesisVideoClient
          .getSignalingChannelEndpoint({
            ChannelARN: channelARN,
            SingleMasterChannelEndpointConfiguration: {
              Protocols: ['HTTPS', 'WSS'],
              Role: Role.VIEWER,
            },
          })
          .promise();

        const endpointsByProtocol = getSignalingChannelEndpointResponse.ResourceEndpointList.reduce(
          (endpoints, endpoint) => {
            endpoints[endpoint.Protocol] = endpoint.ResourceEndpoint;
            return endpoints;
          },
          {}
        );

        // 创建信令客户端
        this.viewerSignalingClient = new SignalingClient({
          channelARN,
          channelEndpoint: endpointsByProtocol.WSS,
          role: Role.VIEWER,
          region: this.region,
          credentials: AWS.config.credentials,
          systemClockOffset: kinesisVideoClient.config.systemClockOffset,
          clientId,
        });

        // 获取 ICE 服务器配置
        const kinesisVideoSignalingChannelsClient = new AWS.KinesisVideoSignalingChannels({
          region: this.region,
          endpoint: endpointsByProtocol.HTTPS,
          credentials: AWS.config.credentials,
          correctClockSkew: true,
        });

        const getIceServerConfigResponse = await kinesisVideoSignalingChannelsClient
          .getIceServerConfig({
            ChannelARN: channelARN,
          })
          .promise();

        const iceServers = [
          { urls: `stun:stun.kinesisvideo.${this.region}.amazonaws.com:443` },
          ...getIceServerConfigResponse.IceServerList.map((iceServer) => ({
            urls: iceServer.Uris,
            username: iceServer.Username,
            credential: iceServer.Password,
          })),
          // TODO: 添加外部TURN服务器（仅用于测试）
          {
            urls: 'turn:27.25.138.55:3478',
            username: 'demo',
            credential: 'HSN95pGqr3QECqWe',
          }
        ];

        console.log('ICE Servers:', iceServers);

        // 创建 PeerConnection
        // TODO: iceTransportPolicy: 'relay'
        this.viewerPeerConnection = new RTCPeerConnection({ iceServers});

        // 监控 ICE 连接状态
        this.viewerPeerConnection.addEventListener('iceconnectionstatechange', () => {
          console.log('Viewer: ICE连接状态变化为:', this.viewerPeerConnection.iceConnectionState);
        });

        // 处理 ICE 候选并添加日志
        this.viewerPeerConnection.addEventListener('icecandidate', (event) => {
          if (event.candidate) {
            console.log('Viewer: 收到 ICE Candidate:', event.candidate.candidate);
            this.viewerSignalingClient.sendIceCandidate(event.candidate);
          } else {
            console.log('Viewer: 所有ICE候选已收集完毕');
          }
        });

        // 处理远程轨道
        this.remoteStream = new MediaStream();
        this.viewerPeerConnection.addEventListener('track', (event) => {
          console.log('Viewer: 收到远程轨道');
          this.remoteStream.addTrack(event.track);
          this.$refs.remoteVideo.srcObject = this.remoteStream;

          // 试图强制播放，以确保视频开始播放
          this.$refs.remoteVideo.play().catch(error => console.error('Remote video play error:', error));
        });

        // 处理信令客户端事件
        this.viewerSignalingClient.on('open', async () => {
          console.log('Viewer: 信令客户端已连接');

          // 创建 SDP Offer
          const offer = await this.viewerPeerConnection.createOffer({
            offerToReceiveAudio: true,
            offerToReceiveVideo: true,
          });
          await this.viewerPeerConnection.setLocalDescription(offer);

          // 发送 SDP Offer
          this.viewerSignalingClient.sendSdpOffer(this.viewerPeerConnection.localDescription);
          console.log('Viewer: 发送 SDP Offer');
        });

        this.viewerSignalingClient.on('sdpAnswer', async (answer) => {
          try {
            console.log('Viewer: 收到 SDP Answer');
            await this.viewerPeerConnection.setRemoteDescription(answer);
          } catch (error) {
            console.error('Viewer: 设置 SDP Answer 失败', error);
          }
        });

        this.viewerSignalingClient.on('iceCandidate', async (candidate) => {
          try {
            console.log('Viewer: 收到 ICE 候选:', candidate.candidate);
            await this.viewerPeerConnection.addIceCandidate(candidate);
          } catch (error) {
            console.error('Viewer: 添加ICE候选失败', error);
          }
        });

        this.viewerSignalingClient.on('close', () => {
          console.log('Viewer: 信令客户端已关闭');
        });

        this.viewerSignalingClient.on('error', (error) => {
          console.log('Viewer: 信令客户端错误：' + error.message);
        });

        // 开启信令客户端
        this.viewerSignalingClient.open();
        console.log('Viewer: 信令客户端正在连接...');
      } catch (error) {
        console.log('Viewer错误：' + error.message);
      }
    }
  }
}
</script>

<style scoped>
.device-actions {
  display: flex;
  align-items: center;
  /* 垂直居中对齐 */
}
</style>