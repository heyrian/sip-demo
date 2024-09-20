<script setup>
import { ref, watch } from 'vue';
import * as SIP from 'sip.js';

const user = ref(null);
const isConnected = ref(false);
const isRegistered = ref(false);
const isInCall = ref(false);
const isMuted = ref(false);
const remoteAudio = ref(null);

// const connect = async () => {
//     const userAgent = new SIP.UserAgent({
//         uri: new SIP.URI('sip', 'userA', 'sipjs.onsip.com'),
//         transportOptions: {
//             server: 'wss://edge.sip.onsip.com'
//         }
//     });
//     user.value = userAgent;
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

// const initiateCall = async () => {
//     if (user.value) {
//         const target = new SIP.URI('sip', 'userB', 'sipjs.onsip.com');
//         const inviter = new SIP.Inviter(user.value, target);
        
//         // 設置媒體處理
//         inviter.stateChange.addListener((newState) => {
//             if (newState === SIP.SessionState.Established) {
//                 const pc = inviter.sessionDescriptionHandler.peerConnection;
//                 pc.getReceivers().forEach((receiver) => {
//                     if (receiver.track.kind === 'audio') {
//                         const stream = new MediaStream([receiver.track]);
//                         if (remoteAudio.value) {
//                             remoteAudio.value.srcObject = stream;
//                         }
//                     }
//                 });
//             }
//         });

//         await inviter.invite();
//         isInCall.value = true;
//     }
// };


const connect = async () => {
    const simpleUser = new SIP.Web.SimpleUser('wss://edge.sip.onsip.com', {
        aor: 'sip:userC@sipjs.onsip.com',
        media: {
            constraints: { audio: true, video: false }
        }
    });
    user.value = simpleUser;
    await simpleUser.connect();
    isConnected.value = true;
};

const register = async () => {
    if (user.value) {
        try {
            await user.value.register();
            isRegistered.value = true;
        } catch (error) {
            console.error('註冊失敗:', error);
            alert('註冊失敗，請稍後再試。');
        }
    }
};

const initiateCall = async () => {
    if (user.value && isRegistered.value) {
        try {
            const target = 'sip:userD@sipjs.onsip.com';
            console.log('開始呼叫:', target);

            // 發起通話
            await user.value.call(target);
            isInCall.value = true;

            // 設置遠端音訊
            user.value.delegate = {
                onCallAnswered: () => {
                    console.log('通話已接通');
                    // 當通話接通時，自動播放音訊
                    if (remoteAudio.value) {
                        remoteAudio.value.play().catch(error => {
                            console.error('自動播放失敗:', error);
                        });
                    }
                },
                onCallHangup: () => {
                    console.log('通話已結束');
                    isInCall.value = false;
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
            console.error('發起通話失敗:', error);
            alert('發起通話失敗，請稍後再試。');
        }
    } else {
        console.error('無法發起通話：用戶未連接或未註冊');
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

</script>

<template>
    <div class="user-a">
        <h3>用戶 A</h3>
        <button @click="connect" :disabled="isConnected">連接</button>
        <button @click="register" :disabled="!isConnected || isRegistered">註冊</button>
        <button @click="initiateCall" :disabled="!isRegistered || isInCall">發起通話</button>
        <button @click="endCall" :disabled="!isInCall">結束通話</button>
        <button @click="unregister" :disabled="!isRegistered">取消註冊</button>
        <button @click="disconnect" :disabled="!isConnected">斷開連接</button>
        <div>
            <input type="checkbox" v-model="isMuted" :disabled="!isInCall"> 靜音
        </div>
        <audio ref="remoteAudio" controls autoplay></audio>
    </div>
</template>