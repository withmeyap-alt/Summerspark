Import React, { useState, useRef, useEffect } from "react";
import SimplePeer from "simple-peer";
import { v4 as uuidv4 } from "uuid";

// CoupleCall 💞 — now with Offline + Online Chat + Avatar Presence!
export default function CoupleCallApp() {
const [name, setName] = useState("You");
const [partnerId, setPartnerId] = useState("Partner");
const [connected, setConnected] = useState(false);
const [inCall, setInCall] = useState(false);
const [room, setRoom] = useState("");
const [cuteMood, setCuteMood] = useState("rose");
const [messages, setMessages] = useState([]);
const [inputMsg, setInputMsg] = useState("");
const offlineQueue = useRef([]);

const localVideo = useRef(null);
const remoteVideo = useRef(null);
const peerRef = useRef(null);
const localStreamRef = useRef(null);
const wsRef = useRef(null);

useEffect(() => {
try {
wsRef.current = new WebSocket("ws://192.168.43.1:4000"); // local hotspot example
wsRef.current.onmessage = (msg) => {
let data;
try {
data = JSON.parse(msg.data);
} catch (e) {
return;
}
if (data.type === "offer") handleRemoteOffer(data.payload);
if (data.type === "answer") handleRemoteAnswer(data.payload);
if (data.type === "candidate") handleRemoteCandidate(data.payload);
if (data.type === "chat") receiveMessage(data.payload);
};
} catch (e) {
console.warn("No signaling server — offline fallback active.");
}
return () => wsRef.current && wsRef.current.close();
}, []);

async function startLocalStream() {
const s = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
localStreamRef.current = s;
if (localVideo.current) localVideo.current.srcObject = s;
return s;
}

function createPeer(initiator = false) {
const p = new SimplePeer({
initiator,
stream: localStreamRef.current || undefined,
trickle: true,
});

p.on("signal", (data) => {  
  const payload = { room, data };  
  if (wsRef.current && wsRef.current.readyState === WebSocket.OPEN) {  
    wsRef.current.send(JSON.stringify({ type: initiator ? "offer" : "answer", payload }));  
  }  
});  

p.on("stream", (remoteStream) => {  
  if (remoteVideo.current) remoteVideo.current.srcObject = remoteStream;  
  setConnected(true);  
  setInCall(true);  
});  

p.on("data", (data) => {  
  try {  
    const msgObj = JSON.parse(data.toString());  
    if (msgObj.type === "chat") receiveMessage(msgObj.payload);  
  } catch (e) {}  
});  

return p;

}

async function startCallAsCaller() {
await startLocalStream();
peerRef.current = createPeer(true);
}

async function answerCall() {
await startLocalStream();
peerRef.current = createPeer(false);
}

function handleRemoteOffer(payload) {
if (!peerRef.current) answerCall();
peerRef.current.signal(payload.data);
}

function handleRemoteAnswer(payload) {
if (peerRef.current) peerRef.current.signal(payload.data);
}

function handleRemoteCandidate(payload) {
if (peerRef.current) peerRef.current.signal(payload.data);
}

function endCall() {
if (peerRef.current) peerRef.current.destroy();
if (localStreamRef.current) localStreamRef.current.getTracks().forEach((t) => t.stop());
setInCall(false);
setConnected(false);
}

// ✨ Chat system (works online or offline)
function sendMessage() {
if (!inputMsg.trim()) return;
const msg = { from: name, text: inputMsg, time: new Date().toLocaleTimeString() };
setMessages((prev) => [...prev, msg]);

const payload = JSON.stringify({ type: "chat", payload: msg });  
if (peerRef.current && connected) {  
  peerRef.current.send(payload);  
} else {  
  // store offline  
  offlineQueue.current.push(msg);  
  localStorage.setItem("offlineMessages", JSON.stringify(offlineQueue.current));  
}  
setInputMsg("");

}

function receiveMessage(msg) {
setMessages((prev) => [...prev, msg]);
}

// sync offline messages once connected
useEffect(() => {
if (connected && offlineQueue.current.length > 0) {
offlineQueue.current.forEach((msg) => {
const payload = JSON.stringify({ type: "chat", payload: msg });
peerRef.current.send(payload);
});
offlineQueue.current = [];
localStorage.removeItem("offlineMessages");
}
}, [connected]);

const themeGradients = {
rose: "from-pink-200 to-rose-300",
lavender: "from-violet-200 to-indigo-200",
mint: "from-emerald-200 to-teal-200",
};

return (
<div className={min-h-screen p-6 bg-gradient-to-b ${themeGradients[cuteMood]} flex items-center justify-center}>
<div className="max-w-md w-full bg-white/80 backdrop-blur rounded-2xl p-6 shadow-2xl">
<h1 className="text-2xl font-bold text-center mb-4">CoupleCall 💞</h1>

{/* Partner Avatar / Presence */}  
    <div className="flex justify-center mb-4">  
      {connected ? (  
        <div className="relative">  
          <div className="w-12 h-12 rounded-full bg-pink-400 border-2 border-white animate-bounce">  
            <span className="absolute inset-0 flex items-center justify-center text-white font-bold">💖</span>  
            <div className="absolute bottom-0 right-0 w-3 h-3 rounded-full bg-green-400 border-2 border-white animate-pulse"></div>  
          </div>  
          <div className="text-xs text-center mt-1">{partnerId}</div>  
        </div>  
      ) : (  
        <div className="w-12 h-12 rounded-full bg-gray-300 border-2 border-white flex items-center justify-center text-gray-500">  
          👤  
        </div>  
      )}  
    </div>  

    {/* Video section */}  
    <div className="grid grid-cols-2 gap-2 mb-4">  
      <video  
        ref={localVideo}  
        autoPlay  
        muted  
        playsInline  
        className="w-full h-40 rounded-xl bg-gray-100 object-cover"  
      />  
      <video  
        ref={remoteVideo}  
        autoPlay  
        playsInline  
        className="w-full h-40 rounded-xl bg-gray-100 object-cover"  
      />  
    </div>  

    {/* Chat section */}  
    <div className="bg-white/60 rounded-xl p-3 mb-3 h-40 overflow-y-auto">  
      {messages.map((m, i) => (  
        <div key={i} className={`my-1 ${m.from === name ? "text-right" : "text-left"}`}>  
          <div className={`inline-block px-3 py-1 rounded-xl ${m.from === name ? "bg-pink-300" : "bg-gray-200"}`}>  
            <span className="text-sm">{m.text}</span>  
          </div>  
          <div className="text-[10px] text-gray-500">{m.time}</div>  
        </div>  
      ))}  
    </div>  

    <div className="flex gap-2 mb-4">  
      <input  
        value={inputMsg}  
        onChange={(e) => setInputMsg(e.target.value)}  
        placeholder="Type a cute message..."  
        className="flex-1 p-2 rounded-xl border"  
      />  
      <button onClick={sendMessage} className="px-3 py-2 bg-pink-500 text-white rounded-xl">  
        💌  
      </button>  
    </div>  

    {/* Controls */}  
    <div className="flex justify-between items-center">  
      <button onClick={startCallAsCaller} className="bg-pink-500 text-white px-4 py-2 rounded-xl">  
        Start Call  
      </button>  
      <button onClick={answerCall} className="bg-white border px-4 py-2 rounded-xl">  
        Answer  
      </button>  
      <button onClick={endCall} className="bg-red-400 text-white px-4 py-2 rounded-xl">  
        End  
      </button>  
    </div>  

    <footer className="mt-3 text-xs text-center text-gray-600">  
      Offline + Online texting is auto-synced 💕 — stay close even when networks fade.  
    </footer>  
  </div>  
</div>

);
}
