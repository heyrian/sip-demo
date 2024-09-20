<script setup>
import { ref } from 'vue';
import * as SIP from 'sip.js';

const user = ref(null);
const registerer = ref(null); 
const isConnected = ref(false);
const isRegistered = ref(false);
const isIncomingCall = ref(false);
const isInCall = ref(false);
const remoteAudio = ref(null);

const connect = async () => {
    const userAgent = new SIP.UserAgent({
        uri: new SIP.URI('sip', 'userB', 'sipjs.onsip.com'),
        transportOptions: {
            server: 'wss://edge.sip.onsip.com'
        }
    });
    user.value = userAgent;
    userAgent.delegate = {
        onInvite: (invitation) => {
            isIncomingCall.value = true;
            user.value.invitation = invitation;

            // 設置媒體處理
            invitation.stateChange.addListener((newState) => {
                if (newState === SIP.SessionState.Established) {
                    const pc = invitation.sessionDescriptionHandler.peerConnection;
                    pc.getReceivers().forEach((receiver) => {
                        if (receiver.track.kind === 'audio') {
                            const stream = new MediaStream([receiver.track]);
                            if (remoteAudio.value) {
                                remoteAudio.value.srcObject = stream;
                            }
                        }
                    });
                }
            });
        }
    };
    await userAgent.start();
    isConnected.value = true;
};


const register = async () => {
    if (user.value) {
        try {
            registerer.value = new SIP.Registerer(user.value);
            await registerer.value.register();
            isRegistered.value = true;
        } catch (error) {
            console.error('註冊失敗:', error);
            alert('註冊失敗，請稍後再試。');
        }
    }
};

const answerCall = async () => {
    if (user.value && user.value.invitation) {
        await user.value.invitation.accept();
        isInCall.value = true;
        isIncomingCall.value = false;
    }
};

const endCall = async () => {
  if (user.value && user.value.invitation) {
    try {
      await user.value.invitation.bye();
      isInCall.value = false;
      isIncomingCall.value = false;
      if (remoteAudio.value) {
        remoteAudio.value.srcObject = null;
      }
    } catch (error) {
      console.error('掛斷電話失敗:', error);
      alert('掛斷電話失敗，請稍後再試。');
    } finally {
      user.value.invitation = null;
    }
  }
};


const unregister = async () => {
    if (registerer.value) {
        try {
            await registerer.value.unregister();
            isRegistered.value = false;
        } catch (error) {
            console.error('取消註冊失敗:', error);
            alert('取消註冊失敗，請稍後再試。');
        } finally {
            registerer.value = null;
        }
    } else {
        console.warn('註冊器不存在，可能尚未註冊');
    }
};

const disconnect = async () => {
    if (user.value) {
      if (registerer.value) {
            await unregister();  // 確保在斷開連接前取消註冊
        }
        await user.value.stop();
        isConnected.value = false;
    }
};


</script>

<template>
    <div class="user-b">
        <h3>用戶 B</h3>
        <button @click="connect" :disabled="isConnected">連接</button>
        <button @click="register" :disabled="!isConnected || isRegistered">註冊</button>
        <button @click="answerCall" :disabled="!isIncomingCall">接聽來電</button>
        <button @click="endCall" :disabled="!isInCall">結束通話</button>
        <button @click="unregister" :disabled="!isRegistered">取消註冊</button>
        <button @click="disconnect" :disabled="!isConnected">斷開連接</button>

        <audio ref="remoteAudio" controls autoplay></audio>
    </div>
</template>