<script setup>
import { ref, watch } from 'vue';
import * as SIP from 'sip.js';

const user = ref(null);
const isConnected = ref(false);
const isRegistered = ref(false);
const isIncomingCall = ref(false);
const isInCall = ref(false);
const isMuted = ref(false);
const remoteAudio = ref(null);

// const connect = async () => {
//     const userAgent = new SIP.UserAgent({
//         uri: new SIP.URI('sip', 'userB', 'sipjs.onsip.com'),
//         transportOptions: {
//             server: 'wss://edge.sip.onsip.com'
//         }
//     });
//     user.value = userAgent;
//     userAgent.delegate = {
//         onInvite: (invitation) => {
//             isIncomingCall.value = true;
//             user.value.invitation = invitation;

//             // 設置媒體處理
//             invitation.stateChange.addListener((newState) => {
//                 if (newState === SIP.SessionState.Established) {
//                     const pc = invitation.sessionDescriptionHandler.peerConnection;
//                     pc.getReceivers().forEach((receiver) => {
//                         if (receiver.track.kind === 'audio') {
//                             const stream = new MediaStream([receiver.track]);
//                             if (remoteAudio.value) {
//                                 remoteAudio.value.srcObject = stream;
//                             }
//                         }
//                     });
//                 }
//             });
//         }
//     };
//     await userAgent.start();
//     isConnected.value = true;
// };


// const register = async () => {
//     if (user.value) {
//         const registerer = new SIP.Registerer(user.value);
//         await registerer.register();
//         isRegistered.value = true;
//     }
// };

// const answerCall = async () => {
//     if (user.value && user.value.invitation) {
//         await user.value.invitation.accept();
//         isInCall.value = true;
//         isIncomingCall.value = false;
//     }
// };

const connect = async () => {
    const simpleUser = new SIP.Web.SimpleUser('wss://edge.sip.onsip.com', {
        aor: 'sip:userD@sipjs.onsip.com',
        media: {
            constraints: { audio: true, video: false },
            remote: { audio: remoteAudio.value }
        }
    });
    user.value = simpleUser;

    // 設置來電處理
    simpleUser.delegate = {
        onCallReceived: () => {
            console.log('收到來電');
            isIncomingCall.value = true;
        }
    };

    await simpleUser.connect();
    isConnected.value = true;
};

const register = async () => {
    if (user.value) {
        try {
            await user.value.register();
            isRegistered.value = true;
            console.log('UserD 註冊成功');
        } catch (error) {
            console.error('註冊失敗:', error);
            alert('註冊失敗，請稍後再試。');
        }
    }
};
const answerCall = async () => {
    if (user.value && isIncomingCall.value) {
        try {
            console.log('嘗試接聽來電');
            await user.value.answer();
            console.log('成功接聽來電');
            isInCall.value = true;
            isIncomingCall.value = false;

            // 設置遠端音訊
            user.value.delegate = {
                onCallHangup: () => {
                    console.log('通話已結束');
                    isInCall.value = false;
                    isIncomingCall.value = false;
                    if (remoteAudio.value) {
                        remoteAudio.value.srcObject = null;
                    }
                },
                onTrack: (track) => {
                    console.log('收到音訊軌道');
                    if (track.kind === 'audio') {
                        const remoteStream = new MediaStream([track]);
                        if (remoteAudio.value) {
                            remoteAudio.value.srcObject = remoteStream;
                        }
                    }
                }
            };
        } catch (error) {
            console.error('接聽來電失敗:', error);
            alert('接聽來電失敗，請稍後再試。');
        }
    } else {
        console.error('無法接聽來電：用戶未連接或沒有來電');
    }
};
// 結束通話函數
const endCall = async () => {
    if (user.value && isInCall.value) {
        try {
            await user.value.hangup();
            isInCall.value = false;
            if (remoteAudio.value) {
                remoteAudio.value.srcObject = null;
            }
        } catch (error) {
            console.error('結束通話失敗:', error);
            alert('結束通話失敗，請稍後再試。');
        }
    }
};

const unregister = async () => {
    if (user.value) {
        try {
            await user.value.unregister();
            isRegistered.value = false;
        } catch (error) {
            console.error('取消註冊失敗:', error);
            alert('取消註冊失敗，請稍後再試。');
        }
    }
};

const disconnect = async () => {
    if (user.value) {
        await user.value.stop();
        isConnected.value = false;
    }
};

watch(isMuted, (newValue) => {
    if (user.value && user.value.session) {
        newValue ? user.value.session.mute() : user.value.session.unmute();
    }
});

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
        <div>
            <input type="checkbox" v-model="isMuted" :disabled="!isInCall"> 靜音
        </div>
        <audio ref="remoteAudio" controls autoplay></audio>
    </div>
</template>