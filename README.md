<!DOCTYPE html>
<html lang="fa" dir="rtl">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
  <meta name="theme-color" content="#14172B" />
  <meta name="apple-mobile-web-app-capable" content="yes" />
  <title>کانون | دستیار مطالعه</title>
  <link href="https://fonts.googleapis.com/css2?family=Vazirmatn:wght@400;500;600;700;800&display=swap" rel="stylesheet" />
  <script crossorigin src="https://unpkg.com/react@18.3.1/umd/react.production.min.js"></script>
  <script crossorigin src="https://unpkg.com/react-dom@18.3.1/umd/react-dom.production.min.js"></script>
  <script src="https://unpkg.com/@babel/standalone@7.26.0/babel.min.js"></script>
  <script src="https://unpkg.com/recharts@2.12.7/umd/Recharts.js"></script>
  <!-- Firebase -->
  <script src="https://www.gstatic.com/firebasejs/10.14.1/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/10.14.1/firebase-auth-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/10.14.1/firebase-firestore-compat.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/html2canvas/1.4.1/html2canvas.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
  <style>
    html, body, #root { margin: 0; padding: 0; min-height: 100%; background: #14172B; }
    * { -webkit-tap-highlight-color: transparent; }
  </style>
</head>
<body>
  <div id="root"></div>
  <script type="text/babel" data-presets="react">
    const { useState, useEffect, useRef, useMemo } = React;

    // ---------- Firebase init ----------
    const firebaseConfig = {
      apiKey: "AIzaSyDchUmIqj6XRZuab-iTXt226ZckrncTKjw",
      authDomain: "kaanoonapp.firebaseapp.com",
      projectId: "kaanoonapp",
      storageBucket: "kaanoonapp.firebasestorage.app",
      messagingSenderId: "302913840241",
      appId: "1:302913840241:web:f47321d28b279411b5a7d0",
      measurementId: "G-68GM6P1P5M"
    };
    if (!firebase.apps.length) {
      firebase.initializeApp(firebaseConfig);
    }
    const db = firebase.firestore();
    const auth = firebase.auth();

    const LEADERBOARD_COLLECTION = "leaderboard";
    const USERS_COLLECTION = "users";

    // ادمین ثابت — فقط تو
    const ADMIN_USERNAME = "admin";
    const ADMIN_PASSWORD = "KanoonAdmin1404";
    const SESSION_KEY = "kanoon:sessionUserId";

    // SHA-256 خالص برای وقتی crypto.subtle در دسترس نیست (مثلاً http روی IP محلی)
    function sha256Fallback(ascii) {
      function rotr(n, x) { return (x >>> n) | (x << (32 - n)); }
      function utf8(str) {
        const out = [];
        for (let i = 0; i < str.length; i++) {
          let c = str.charCodeAt(i);
          if (c < 0x80) out.push(c);
          else if (c < 0x800) out.push(0xc0 | (c >> 6), 0x80 | (c & 0x3f));
          else if (c < 0xd800 || c >= 0xe000) out.push(0xe0 | (c >> 12), 0x80 | ((c >> 6) & 0x3f), 0x80 | (c & 0x3f));
          else {
            i++;
            c = 0x10000 + (((c & 0x3ff) << 10) | (str.charCodeAt(i) & 0x3ff));
            out.push(0xf0 | (c >> 18), 0x80 | ((c >> 12) & 0x3f), 0x80 | ((c >> 6) & 0x3f), 0x80 | (c & 0x3f));
          }
        }
        return out;
      }
      const K = [
        0x428a2f98,0x71374491,0xb5c0fbcf,0xe9b5dba5,0x3956c25b,0x59f111f1,0x923f82a4,0xab1c5ed5,
        0xd807aa98,0x12835b01,0x243185be,0x550c7dc3,0x72be5d74,0x80deb1fe,0x9bdc06a7,0xc19bf174,
        0xe49b69c1,0xefbe4786,0x0fc19dc6,0x240ca1cc,0x2de92c6f,0x4a7484aa,0x5cb0a9dc,0x76f988da,
        0x983e5152,0xa831c66d,0xb00327c8,0xbf597fc7,0xc6e00bf3,0xd5a79147,0x06ca6351,0x14292967,
        0x27b70a85,0x2e1b2138,0x4d2c6dfc,0x53380d13,0x650a7354,0x766a0abb,0x81c2c92e,0x92722c85,
        0xa2bfe8a1,0xa81a664b,0xc24b8b70,0xc76c51a3,0xd192e819,0xd6990624,0xf40e3585,0x106aa070,
        0x19a4c116,0x1e376c08,0x2748774c,0x34b0bcb5,0x391c0cb3,0x4ed8aa4a,0x5b9cca4f,0x682e6ff3,
        0x748f82ee,0x78a5636f,0x84c87814,0x8cc70208,0x90befffa,0xa4506ceb,0xbef9a3f7,0xc67178f2
      ];
      const bytes = utf8(ascii);
      const l = bytes.length;
      bytes.push(0x80);
      while (bytes.length % 64 !== 56) bytes.push(0);
      const bitLen = l * 8;
      for (let i = 7; i >= 0; i--) bytes.push((bitLen / Math.pow(2, i * 8)) & 0xff);
      let [h0,h1,h2,h3,h4,h5,h6,h7] = [0x6a09e667,0xbb67ae85,0x3c6ef372,0xa54ff53a,0x510e527f,0x9b05688c,0x1f83d9ab,0x5be0cd19];
      for (let i = 0; i < bytes.length; i += 64) {
        const w = new Array(64);
        for (let j = 0; j < 16; j++) {
          const o = i + j * 4;
          w[j] = ((bytes[o] << 24) | (bytes[o+1] << 16) | (bytes[o+2] << 8) | bytes[o+3]) >>> 0;
        }
        for (let j = 16; j < 64; j++) {
          const s0 = rotr(7, w[j-15]) ^ rotr(18, w[j-15]) ^ (w[j-15] >>> 3);
          const s1 = rotr(17, w[j-2]) ^ rotr(19, w[j-2]) ^ (w[j-2] >>> 10);
          w[j] = (w[j-16] + s0 + w[j-7] + s1) >>> 0;
        }
        let [a,b,c,d,e,f,g,h] = [h0,h1,h2,h3,h4,h5,h6,h7];
        for (let j = 0; j < 64; j++) {
          const S1 = rotr(6, e) ^ rotr(11, e) ^ rotr(25, e);
          const ch = (e & f) ^ (~e & g);
          const t1 = (h + S1 + ch + K[j] + w[j]) >>> 0;
          const S0 = rotr(2, a) ^ rotr(13, a) ^ rotr(22, a);
          const maj = (a & b) ^ (a & c) ^ (b & c);
          const t2 = (S0 + maj) >>> 0;
          h = g; g = f; f = e; e = (d + t1) >>> 0;
          d = c; c = b; b = a; a = (t1 + t2) >>> 0;
        }
        h0 = (h0 + a) >>> 0; h1 = (h1 + b) >>> 0; h2 = (h2 + c) >>> 0; h3 = (h3 + d) >>> 0;
        h4 = (h4 + e) >>> 0; h5 = (h5 + f) >>> 0; h6 = (h6 + g) >>> 0; h7 = (h7 + h) >>> 0;
      }
      return [h0,h1,h2,h3,h4,h5,h6,h7].map((x) => ("00000000" + x.toString(16)).slice(-8)).join("");
    }

    async function hashPassword(password) {
      const input = "kanoon|" + password;
      try {
        if (globalThis.crypto && crypto.subtle && typeof crypto.subtle.digest === "function") {
          const data = new TextEncoder().encode(input);
          const buf = await crypto.subtle.digest("SHA-256", data);
          return Array.from(new Uint8Array(buf)).map((b) => b.toString(16).padStart(2, "0")).join("");
        }
      } catch (e) { /* fallback */ }
      return sha256Fallback(input);
    }

    async function findUserByUsername(username) {
      const un = String(username).trim().toLowerCase();
      const snap = await db.collection(USERS_COLLECTION).where("username", "==", un).limit(1).get();
      if (snap.empty) return null;
      const doc = snap.docs[0];
      return { id: doc.id, ...doc.data() };
    }

    async function registerUser(username, password, displayName, grade) {
      const un = String(username).trim().toLowerCase();
      if (!un || un.length < 3) throw new Error("یوزرنیم حداقل ۳ حرف باشد");
      if (!password || password.length < 6) throw new Error("رمز حداقل ۶ حرف باشد");
      if (un === ADMIN_USERNAME) throw new Error("این یوزرنیم رزرو شده است");
      const existing = await findUserByUsername(un);
      if (existing) throw new Error("این یوزرنیم قبلاً ثبت شده");
      const newId = Date.now().toString(36) + Math.random().toString(36).slice(2, 8);
      const passwordHash = await hashPassword(password);
      const profile = {
        userId: newId,
        username: un,
        name: (displayName || "").trim() || un,
        grade: grade || "دوازدهم",
        role: "user",
        passwordHash,
        joinedAt: Date.now(),
      };
      await db.collection(USERS_COLLECTION).doc(newId).set(profile);
      localStorage.setItem(SESSION_KEY, newId);
      const { passwordHash: _, ...safe } = profile;
      return safe;
    }

    async function loginUser(username, password) {
      const un = String(username).trim().toLowerCase();
      // ادمین ثابت
      if (un === ADMIN_USERNAME && password === ADMIN_PASSWORD) {
        const profile = {
          userId: "admin-root",
          username: ADMIN_USERNAME,
          name: "ادمین",
          grade: "—",
          role: "admin",
          joinedAt: Date.now(),
        };
        await db.collection(USERS_COLLECTION).doc("admin-root").set({ ...profile, passwordHash: await hashPassword(password) }, { merge: true });
        localStorage.setItem(SESSION_KEY, "admin-root");
        return profile;
      }
      const found = await findUserByUsername(un);
      if (!found) throw new Error("کاربری با این یوزرنیم پیدا نشد");
      const passwordHash = await hashPassword(password);
      if (found.passwordHash !== passwordHash) throw new Error("یوزرنیم یا رمز اشتباه است");
      localStorage.setItem(SESSION_KEY, found.userId || found.id);
      const { passwordHash: _, ...safe } = found;
      if (!safe.userId) safe.userId = found.id;
      return safe;
    }

    async function logoutUser() {
      localStorage.removeItem(SESSION_KEY);
      try { await auth.signOut(); } catch (e) { /* ignore */ }
    }

    async function restoreSession() {
      const sid = localStorage.getItem(SESSION_KEY);
      if (!sid) return null;
      const doc = await db.collection(USERS_COLLECTION).doc(sid).get();
      if (!doc.exists) {
        localStorage.removeItem(SESSION_KEY);
        return null;
      }
      const data = doc.data();
      const { passwordHash: _, ...safe } = data;
      if (!safe.userId) safe.userId = doc.id;
      return safe;
    }

    async function deleteUserAsAdmin(targetUserId) {
      await db.collection(LEADERBOARD_COLLECTION).doc(targetUserId).delete().catch(() => {});
      await db.collection(USERS_COLLECTION).doc(targetUserId).delete().catch(() => {});
    }

    async function adminResetPassword(targetUserId, newPassword) {
      if (!newPassword || newPassword.length < 6) throw new Error("رمز جدید حداقل ۶ حرف باشد");
      const passwordHash = await hashPassword(newPassword);
      await db.collection(USERS_COLLECTION).doc(targetUserId).set({ passwordHash }, { merge: true });
    }

    async function listAllUsers() {
      const snap = await db.collection(USERS_COLLECTION).get();
      const list = [];
      snap.forEach((d) => {
        const data = d.data();
        const { passwordHash, ...safe } = data;
        if (!safe.userId) safe.userId = d.id;
        list.push(safe);
      });
      return list;
    }


    const TASKS_COLLECTION = "sharedTasks";
    const TASK_PROGRESS_COLLECTION = "taskProgress";

    async function fetchSharedTasks() {
      const snap = await db.collection(TASKS_COLLECTION).get();
      const list = [];
      snap.forEach((d) => list.push({ id: d.id, ...d.data() }));
      list.sort((a, b) => (b.createdAt || 0) - (a.createdAt || 0));
      return list;
    }

    async function createSharedTask(task) {
      const ref = db.collection(TASKS_COLLECTION).doc(task.id);
      await ref.set(task);
      return task;
    }

    async function deleteSharedTask(taskId) {
      await db.collection(TASKS_COLLECTION).doc(taskId).delete();
    }

    async function fetchUserTaskProgress(userId) {
      if (!userId) return {};
      const snap = await db.collection(TASK_PROGRESS_COLLECTION).doc(userId).collection("items").get();
      const map = {};
      snap.forEach((d) => { map[d.id] = d.data(); });
      return map;
    }

    async function saveUserTaskProgress(userId, taskId, progress) {
      if (!userId || !taskId) return;
      await db.collection(TASK_PROGRESS_COLLECTION).doc(userId).collection("items").doc(taskId).set({
        ...progress,
        taskId,
        userId,
        updatedAt: Date.now(),
      }, { merge: true });
    }

    async function publishToLeaderboard(profile, sessions) {
      if (!profile || !profile.userId) return;
      // ادمین وارد لیست رقابت نمی‌شود
      if (profile.role === "admin" || profile.username === "admin") return;
      try {
        const dailyStats = buildDailyStats(sessions);
        await db.collection(LEADERBOARD_COLLECTION).doc(profile.userId).set({
          userId: profile.userId,
          name: profile.name,
          grade: profile.grade,
          username: profile.username || "",
          role: profile.role || "user",
          dailyStats,
          updatedAt: Date.now(),
        }, { merge: true });
      } catch (e) {
        console.error("Firebase publish error:", e);
      }
    }

    async function fetchLeaderboardFromFirebase() {
      const snap = await db.collection(LEADERBOARD_COLLECTION).get();
      const rows = [];
      snap.forEach((doc) => {
        const data = doc.data();
        if (!data || !data.userId) return;
        // ادمین در رقابت نمایش داده نمی‌شود
        if (data.role === "admin" || data.username === "admin" || data.name === "ادمین") return;
        rows.push(data);
      });
      return rows;
    }

    const USER_STUDY_COLLECTION = "userStudy";

    async function pushUserStudy(userId, payload) {
      if (!userId || userId === "admin-root") return;
      try {
        await db.collection(USER_STUDY_COLLECTION).doc(userId).set({
          sessions: payload.sessions || [],
          boxes: payload.boxes || [],
          settings: payload.settings || null,
          dayOverrides: payload.dayOverrides || {},
          weeklyPlan: payload.weeklyPlan || { title: "برنامه مطالعاتی هفتگی", items: [] },
          exams: payload.exams || [],
          updatedAt: Date.now(),
        });
      } catch (e) {
        console.error("pushUserStudy error:", e);
      }
    }

    async function pullUserStudy(userId) {
      if (!userId) return null;
      try {
        const doc = await db.collection(USER_STUDY_COLLECTION).doc(userId).get();
        if (!doc.exists) return null;
        return doc.data();
      } catch (e) {
        console.error("pullUserStudy error:", e);
        return null;
      }
    }

    function mergeSessions(localList, remoteList) {
      const map = {};
      (localList || []).forEach((s) => { if (s && s.id) map[s.id] = s; });
      (remoteList || []).forEach((s) => {
        if (!s || !s.id) return;
        if (!map[s.id] || (s.timestamp || 0) >= (map[s.id].timestamp || 0)) map[s.id] = s;
      });
      return Object.values(map).sort((a, b) => (b.timestamp || 0) - (a.timestamp || 0));
    }

    function mergeBoxes(localList, remoteList) {
      const map = {};
      (localList || []).forEach((b) => { if (b && b.id) map[b.id] = b; });
      (remoteList || []).forEach((b) => {
        if (!b || !b.id) return;
        if (!map[b.id] || (b.createdAt || 0) >= (map[b.id].createdAt || 0)) map[b.id] = b;
      });
      return Object.values(map);
    }

    async function resetLeaderboard() {
      const snap = await db.collection(LEADERBOARD_COLLECTION).get();
      const batch = db.batch();
      let count = 0;
      snap.forEach((doc) => {
        batch.delete(doc.ref);
        count += 1;
      });
      if (count > 0) await batch.commit();
      return count;
    }

const iconProps = (props) => ({
  width: props.size || 24, height: props.size || 24,
  viewBox: "0 0 24 24", fill: "none", stroke: props.color || "currentColor",
  strokeWidth: props.strokeWidth || 2, strokeLinecap: "round", strokeLinejoin: "round",
  className: props.className, style: props.style
});
function createIcon(paths) {
  return function Icon(props) {
    return React.createElement("svg", iconProps(props), ...paths.map((d, i) =>
      typeof d === "string" ? React.createElement("path", { d, key: i }) :
      React.createElement(d[0], { ...d[1], key: i })
    ));
  };
}
const Box = createIcon([["path",{d:"M21 8a2 2 0 0 0-1-1.73l-7-4a2 2 0 0 0-2 0l-7 4A2 2 0 0 0 3 8v8a2 2 0 0 0 1 1.73l7 4a2 2 0 0 0 2 0l7-4A2 2 0 0 0 21 16Z"}],["path",{d:"m3.3 7 8.7 5 8.7-5"}],["path",{d:"M12 22V12"}]]);
const Plus = createIcon([["path",{d:"M5 12h14"}],["path",{d:"M12 5v14"}]]);
const Trash2 = createIcon([["path",{d:"M3 6h18"}],["path",{d:"M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"}],["path",{d:"M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"}],["line",{x1:"10",x2:"10",y1:"11",y2:"17"}],["line",{x1:"14",x2:"14",y1:"11",y2:"17"}]]);
const X = createIcon([["path",{d:"M18 6 6 18"}],["path",{d:"m6 6 12 12"}]]);
const Flame = createIcon([["path",{d:"M8.5 14.5A2.5 2.5 0 0 0 11 12c0-1.38-.5-2-1-3-1.072-2.143-.224-4.054 2-6 .5 2.5 2 4.9 4 6.5 2 1.6 3 3.5 3 5.5a7 7 0 1 1-14 0c0-1.153.433-2.294 1-3a2.5 2.5 0 0 0 2.5 2.5z"}]]);
const Clock3 = createIcon([["circle",{cx:"12",cy:"12",r:"10"}],["path",{d:"M12 6v6h4"}]]);
const BarChart3 = createIcon([["path",{d:"M3 3v18h18"}],["path",{d:"M18 17V9"}],["path",{d:"M13 17V5"}],["path",{d:"M8 17v-3"}]]);
const CheckCircle2 = createIcon([["circle",{cx:"12",cy:"12",r:"10"}],["path",{d:"m9 12 2 2 4-4"}]]);
const Circle = createIcon([["circle",{cx:"12",cy:"12",r:"10"}]]);
const ClipboardList = createIcon([["rect",{width:"8",height:"4",x:"8",y:"2",rx:"1",ry:"1"}],["path",{d:"M16 4h2a2 2 0 0 1 2 2v14a2 2 0 0 1-2 2H6a2 2 0 0 1-2-2V6a2 2 0 0 1 2-2h2"}],["path",{d:"M12 11h4"}],["path",{d:"M12 16h4"}],["path",{d:"M8 11h.01"}],["path",{d:"M8 16h.01"}]]);
const ChevronRight = createIcon([["path",{d:"m9 18 6-6-6-6"}]]);
const ChevronLeft = createIcon([["path",{d:"m15 18-6-6 6-6"}]]);
const Trophy = createIcon([["path",{d:"M6 9H4.5a2.5 2.5 0 0 1 0-5H6"}],["path",{d:"M18 9h1.5a2.5 2.5 0 0 0 0-5H18"}],["path",{d:"M4 22h16"}],["path",{d:"M10 14.66V17c0 .55-.47.98-.97 1.21C7.85 18.75 7 20.24 7 22"}],["path",{d:"M14 14.66V17c0 .55.47.98.97 1.21C16.15 18.75 17 20.24 17 22"}],["path",{d:"M18 2H6v7a6 6 0 0 0 12 0V2Z"}]]);
const Pencil = createIcon([["path",{d:"M17 3a2.85 2.83 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5Z"}],["path",{d:"m15 5 4 4"}]]);
const RefreshCw = createIcon([["path",{d:"M3 12a9 9 0 0 1 9-9 9.75 9.75 0 0 1 6.74 2.74L21 8"}],["path",{d:"M21 3v5h-5"}],["path",{d:"M21 12a9 9 0 0 1-9 9 9.75 9.75 0 0 1-6.74-2.74L3 16"}],["path",{d:"M8 16H3v5"}]]);
const Medal = createIcon([["path",{d:"M7.21 15 2.66 7.6a2 2 0 0 1 .13-2.2L4.4 2.8A2 2 0 0 1 6 2h12a2 2 0 0 1 1.6.8l1.6 2.6a2 2 0 0 1 .14 2.2L16.79 15"}],["path",{d:"M11 12 5.12 2.2"}],["path",{d:"m13 12 5.88-9.8"}],["path",{d:"M8 7h8"}],["circle",{cx:"12",cy:"17",r:"5"}],["path",{d:"M12 18v-2h-.5"}]]);
const Star = createIcon([["polygon",{points:"12 2 15.09 8.26 22 9.27 17 14.14 18.18 21.02 12 17.77 5.82 21.02 7 14.14 2 9.27 8.91 8.26 12 2"}]]);
const Check = createIcon([["path",{d:"M20 6 9 17l-5-5"}]]);
const Minus = createIcon([["path",{d:"M5 12h14"}]]);
const FileText = createIcon([["path",{d:"M15 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V7Z"}],["path",{d:"M14 2v4a2 2 0 0 0 2 2h4"}],["path",{d:"M10 9H8"}],["path",{d:"M16 13H8"}],["path",{d:"M16 17H8"}]]);
const Calendar = createIcon([["path",{d:"M8 2v4"}],["path",{d:"M16 2v4"}],["rect",{width:"18",height:"18",x:"3",y:"4",rx:"2"}],["path",{d:"M3 10h18"}]]);
const Users = createIcon([["path",{d:"M16 21v-2a4 4 0 0 0-4-4H6a4 4 0 0 0-4 4v2"}],["circle",{cx:"9",cy:"7",r:"4"}],["path",{d:"M22 21v-2a4 4 0 0 0-3-3.87"}],["path",{d:"M16 3.13a4 4 0 0 1 0 7.75"}]]);
const { ResponsiveContainer, BarChart, Bar, XAxis, YAxis, Tooltip, CartesianGrid, AreaChart, Area, Cell, ComposedChart, Line } = window.Recharts || {};


// ---------- design tokens ----------
const COLORS = {
  bg: "#14172B",
  surface: "#1E2242",
  surface2: "#262B52",
  border: "#363B66",
  accent: "#E8A33D",
  accentSoft: "#F4C878",
  accent2: "#57C7B8",
  text: "#F2EFE6",
  muted: "#8B90B3",
  danger: "#E2665A",
  silver: "#B8BFDC",
  bronze: "#D0946A",
};

const SUBJECT_PALETTE = ["#E8A33D", "#57C7B8", "#E2665A", "#8C7AE6", "#4FA8E0", "#E67AA6", "#7FCB6B", "#D8A0F0"];

// study-target rules
const WEEKLY_GOAL_HOURS = 43;
const DEFAULT_NORMAL_DAY_HOURS = 5;
const DEFAULT_HOLIDAY_DAY_HOURS = 8;
// JS Date.getDay(): 0=یکشنبه ... 5=جمعه ... 6=شنبه. روزهای تعطیل کاربر: جمعه(5) و یکشنبه(0)
const isHoliday = (date) => { const d = date.getDay(); return d === 5 || d === 0; };
// settings: { normalHours, holidayHours }, dayOverrides: { "YYYY-MM-DD": hours }
function dayTargetMinutes(date, settings = {}, dayOverrides = {}) {
  const key = todayKey(date);
  if (dayOverrides[key] != null && dayOverrides[key] !== "") {
    return Number(dayOverrides[key]) * 60;
  }
  const normalH = settings.normalHours ?? DEFAULT_NORMAL_DAY_HOURS;
  const holidayH = settings.holidayHours ?? DEFAULT_HOLIDAY_DAY_HOURS;
  return (isHoliday(date) ? holidayH : normalH) * 60;
}

const WEEKDAY_FULL_FA = ["یکشنبه", "دوشنبه", "سه‌شنبه", "چهارشنبه", "پنجشنبه", "جمعه", "شنبه"];
const JALALI_MONTHS = ["فروردین", "اردیبهشت", "خرداد", "تیر", "مرداد", "شهریور", "مهر", "آبان", "آذر", "دی", "بهمن", "اسفند"];

// ---------- curriculum (رشته ریاضی فیزیک) ----------
const GROUP_ORDER = ["دهم", "یازدهم", "دوازدهم", "عمومی", "ویژه"];
const CURRICULUM_RAW = [
  { id: "riazi10", name: "ریاضی (۱)", groupLabel: "دهم" },
  { id: "handase10", name: "هندسه (۱)", groupLabel: "دهم" },
  { id: "fizik10", name: "فیزیک (۱)", groupLabel: "دهم" },
  { id: "shimi10", name: "شیمی (۱)", groupLabel: "دهم" },
  { id: "hesaban11", name: "حسابان (۱)", groupLabel: "یازدهم" },
  { id: "handase11", name: "هندسه (۲)", groupLabel: "یازدهم" },
  { id: "amar11", name: "آمار و احتمال", groupLabel: "یازدهم" },
  { id: "fizik11", name: "فیزیک (۲)", groupLabel: "یازدهم" },
  { id: "shimi11", name: "شیمی (۲)", groupLabel: "یازدهم" },
  { id: "hesaban12", name: "حسابان (۲)", groupLabel: "دوازدهم" },
  { id: "handase12", name: "هندسه (۳)", groupLabel: "دوازدهم" },
  { id: "gosaste12", name: "ریاضیات گسسته", groupLabel: "دوازدهم" },
  { id: "fizik12", name: "فیزیک (۳)", groupLabel: "دوازدهم" },
  { id: "shimi12", name: "شیمی (۳)", groupLabel: "دوازدهم" },
  { id: "farsi", name: "فارسی", groupLabel: "عمومی" },
  { id: "dini", name: "دینی", groupLabel: "عمومی" },
  { id: "arabi", name: "عربی", groupLabel: "عمومی" },
  { id: "zaban", name: "زبان", groupLabel: "عمومی" },
  { id: "hoviyat", name: "هویت اجتماعی", groupLabel: "عمومی" },
  { id: "salamat", name: "سلامت و بهداشت", groupLabel: "عمومی" },
  { id: "azmoon_jame", name: "آزمون جامع", groupLabel: "ویژه" },
  { id: "tahlil_azmoon", name: "تحلیل آزمون", groupLabel: "ویژه" },
  { id: "jobrani", name: "جبرانی", groupLabel: "ویژه" },
];
const CURRICULUM = CURRICULUM_RAW.map((c, i) => ({ ...c, color: SUBJECT_PALETTE[i % SUBJECT_PALETTE.length] }));
const curriculumById = (id) => CURRICULUM.find((c) => c.id === id);

// فصل‌های دروس (بر اساس کتاب‌های رسمی رشته ریاضی + عمومی)
const SUBJECT_CHAPTERS = {
  // —— اختصاصی دهم (ریاضی فیزیک) ——
  riazi10: [
    { num: 1, title: "مجموعه، الگو و دنباله" },
    { num: 2, title: "مثلثات" },
    { num: 3, title: "توان‌های گویا و عبارت‌های جبری" },
    { num: 4, title: "معادله‌ها و نامعادله‌ها" },
    { num: 5, title: "تابع" },
    { num: 6, title: "شمارش، بدون شمردن" },
    { num: 7, title: "آمار و احتمال" },
  ],
  handase10: [
    { num: 1, title: "ترسیم‌های هندسی و استدلال" },
    { num: 2, title: "قضیه تالس، تشابه و کاربردهای آن" },
    { num: 3, title: "چندضلعی‌ها" },
    { num: 4, title: "تجسم فضایی" },
  ],
  fizik10: [
    { num: 1, title: "فیزیک و اندازه‌گیری" },
    { num: 2, title: "ویژگی‌های فیزیکی مواد" },
    { num: 3, title: "کار، انرژی و توان" },
    { num: 4, title: "دما و گرما" },
  ],
  shimi10: [
    { num: 1, title: "کیهان زادگاه عناصر" },
    { num: 2, title: "ردپای گازها در زندگی" },
    { num: 3, title: "آب، آهنگ زندگی" },
  ],
  // —— اختصاصی یازدهم ——
  hesaban11: [
    { num: 1, title: "جبر و معادله" },
    { num: 2, title: "تابع" },
    { num: 3, title: "توابع نمایی و لگاریتمی" },
    { num: 4, title: "مثلثات" },
    { num: 5, title: "حد و پیوستگی" },
  ],
  handase11: [
    { num: 1, title: "دایره" },
    { num: 2, title: "تبدیل‌های هندسی و کاربردها" },
    { num: 3, title: "روابط طولی در مثلث" },
  ],
  amar11: [
    { num: 1, title: "آشنایی با مبانی ریاضیات" },
    { num: 2, title: "احتمال" },
    { num: 3, title: "آمار توصیفی" },
    { num: 4, title: "آمار استنباطی" },
  ],
  fizik11: [
    { num: 1, title: "الکتریسیته ساکن" },
    { num: 2, title: "جریان الکتریکی و مدارهای جریان مستقیم" },
    { num: 3, title: "مغناطیس و القای الکترومغناطیسی" },
  ],
  shimi11: [
    { num: 1, title: "قدر هدایای زمینی را بدانیم" },
    { num: 2, title: "در پی غذای سالم" },
    { num: 3, title: "پوشاک، نیازی پایان‌ناپذیر" },
  ],
  // —— اختصاصی دوازدهم ——
  hesaban12: [
    { num: 1, title: "تابع" },
    { num: 2, title: "مثلثات" },
    { num: 3, title: "حدهای نامتناهی و حد در بی‌نهایت" },
    { num: 4, title: "مشتق" },
    { num: 5, title: "کاربردهای مشتق" },
  ],
  handase12: [
    { num: 1, title: "ماتریس و کاربردها" },
    { num: 2, title: "آشنایی با مقاطع مخروطی" },
    { num: 3, title: "بردارها" },
  ],
  gosaste12: [
    { num: 1, title: "آشنایی با نظریه اعداد" },
    { num: 2, title: "گراف و مدل‌سازی" },
    { num: 3, title: "ترکیبیات (شمارش)" },
  ],
  fizik12: [
    { num: 1, title: "حرکت بر خط راست" },
    { num: 2, title: "دینامیک و حرکت دایره‌ای" },
    { num: 3, title: "نوسان و موج" },
    { num: 4, title: "برهم‌کنش‌های موج" },
    { num: 5, title: "آشنایی با فیزیک اتمی" },
    { num: 6, title: "فیزیک هسته‌ای" },
  ],
  shimi12: [
    { num: 1, title: "مولکول‌ها در خدمت تندرستی" },
    { num: 2, title: "آسایش و رفاه در سایه شیمی" },
    { num: 3, title: "شیمی جلوه‌ای از هنر، زیبایی و ماندگاری" },
    { num: 4, title: "شیمی، راهی به‌سوی آینده‌ای روشن‌تر" },
  ],
  // —— عمومی فقط دوازدهم ——
  // فارسی: بر اساس درس‌های کتاب فارسی ۳
  farsi: [
    { num: 1, title: "درس ۱ — شکر نعمت" },
    { num: 2, title: "درس ۲ — مست و هشیار" },
    { num: 3, title: "درس ۳ — آزادی" },
    { num: 4, title: "درس ۴ — درس آزاد (ادبیات بومی)" },
    { num: 5, title: "درس ۵ — دماوندیه" },
    { num: 6, title: "درس ۶ — نی‌نامه" },
    { num: 7, title: "درس ۷ — در حقیقت عشق" },
    { num: 8, title: "درس ۸ — از پاریز تا پاریس" },
    { num: 9, title: "درس ۹ — کویر" },
    { num: 10, title: "درس ۱۰ — فصل شکوفایی" },
    { num: 11, title: "درس ۱۱ — آن شب عزیز" },
    { num: 12, title: "درس ۱۲ — گذر سیاوش از آتش" },
    { num: 13, title: "درس ۱۳ — خوان هشتم" },
    { num: 14, title: "درس ۱۴ — سی مرغ و سیمرغ" },
    { num: 15, title: "درس ۱۵ — درس آزاد" },
    { num: 16, title: "درس ۱۶ — کباب غاز" },
    { num: 17, title: "درس ۱۷ — خندهٔ تو" },
    { num: 18, title: "درس ۱۸ — عشق جاودانی" },
  ],

  dini: [
    { num: 1, title: "درس ۱ — هستی‌بخش" },
    { num: 2, title: "درس ۲ — یگانه‌ی بی‌همتا" },
    { num: 3, title: "درس ۳ — توحید و سبک زندگی" },
    { num: 4, title: "درس ۴ — فقط برای او" },
    { num: 5, title: "درس ۵ — قدرت پرواز" },
    { num: 6, title: "درس ۶ — پیروز حقیقت" },
    { num: 7, title: "درس ۷ — دل در گرو حق" },
    { num: 8, title: "درس ۸ — بازگشت" },
    { num: 9, title: "درس ۹ — آینده‌ی روشن" },
    { num: 10, title: "درس ۱۰ — آهنگ سفر" },
  ],
  arabi: [
    { num: 1, title: "درس ۱ — الدّینُ و التَّدَیُّن" },
    { num: 2, title: "درس ۲ — في مَحْضَرِ المُعَلِّم" },
    { num: 3, title: "درس ۳ — عَجائِبُ الْأَشْياء" },
    { num: 4, title: "درس ۴ — أَدَبُ الْعَيْشِ وَ الْعِشْرَة" },
  ],
  zaban: [
    { num: 1, title: "Lesson 1 — Sense of Appreciation" },
    { num: 2, title: "Lesson 2 — Look It Up!" },
    { num: 3, title: "Lesson 3 — Renewable Energy" },
  ],
  hoviyat: [
    { num: 1, title: "درس ۱ — کنش اجتماعی" },
    { num: 2, title: "درس ۲ — پدیده‌های اجتماعی" },
    { num: 3, title: "درس ۳ — جامعه و فرهنگ (۱)" },
    { num: 4, title: "درس ۴ — جامعه و فرهنگ (۲)" },
    { num: 5, title: "درس ۵ — هویت" },
    { num: 6, title: "درس ۶ — باز تولید هویت اجتماعی" },
    { num: 7, title: "درس ۷ — تغییرات هویتی جامعه" },
    { num: 8, title: "درس ۸ — هویت ایرانی" },
    { num: 9, title: "درس ۹ — ابعاد فرهنگی، سیاسی، اقتصادی و جغرافیایی هویت ایرانی" },
    { num: 10, title: "درس ۱۰ — اقتدار فرهنگی، سیاسی و اقتصادی" },
  ],
  salamat: [
    { num: 1, title: "درس ۱ — سلامت چیست؟" },
    { num: 2, title: "درس ۲ — سبک زندگی" },
    { num: 3, title: "درس ۳ — برنامهٔ غذایی سالم" },
    { num: 4, title: "درس ۴ — کنترل وزن و تناسب اندام" },
    { num: 5, title: "درس ۵ — بهداشت و ایمنی مواد غذایی" },
    { num: 6, title: "درس ۶ — بیماری‌های غیرواگیر" },
    { num: 7, title: "درس ۷ — بیماری‌های واگیر" },
    { num: 8, title: "درس ۸ — بهداشت فردی" },
    { num: 9, title: "درس ۹ — بهداشت ازدواج و باروری" },
    { num: 10, title: "درس ۱۰ — بهداشت روان" },
    { num: 11, title: "درس ۱۱ — مصرف دخانیات و الکل" },
    { num: 12, title: "درس ۱۲ — اعتیاد به مواد مخدر و عوارض آن" },
    { num: 13, title: "درس ۱۳ — پیشگیری از اختلالات اسکلتی–عضلانی" },
    { num: 14, title: "درس ۱۴ — پیشگیری از حوادث خانگی" },
  ],
  azmoon_jame: [{ num: 1, title: "عمومی" }],
  tahlil_azmoon: [{ num: 1, title: "عمومی" }],
  jobrani: [{ num: 1, title: "عمومی" }],
};

function chaptersOf(subjectId) {
  return SUBJECT_CHAPTERS[subjectId] || [{ num: 1, title: "عمومی" }];
}

function isLessonBasedSubject(subjectId) {
  return ["farsi", "dini", "arabi", "zaban", "hoviyat", "salamat"].includes(subjectId);
}

function chapterLabel(subjectId, chapterNum) {
  const ch = chaptersOf(subjectId).find((c) => c.num === Number(chapterNum));
  if (!ch) {
    if (chapterNum == null) return "";
    return isLessonBasedSubject(subjectId) ? `درس ${toFa(chapterNum)}` : `فصل ${toFa(chapterNum)}`;
  }
  // عنوان‌ها خودشان اغلب شماره درس دارند؛ همان را نشان بده
  return ch.title;
}

function percentFromAnswers(correct, wrong, total) {
  const t = Number(total) || 0;
  if (t <= 0) return null;
  const c = Number(correct) || 0;
  const w = Number(wrong) || 0;
  const p = ((c * 3 - w) / (t * 3)) * 100;
  return Math.round(Math.max(0, Math.min(100, p)) * 10) / 10;
}

/** درصد اگر غلط صفر باشد (فقط برای آزمون درصدی) */
function percentIfNoWrong(correct, total) {
  const t = Number(total) || 0;
  if (t <= 0) return null;
  const c = Number(correct) || 0;
  return Math.round(Math.max(0, Math.min(100, (c / t) * 100)) * 10) / 10;
}

/** آیا آزمون درصدی است؟ (نه نمره‌ای/خام) */
function isPercentExam(ex) {
  const rows = (normalizeExam(ex)?.subjects) || [];
  if (!rows.length) return true;
  // اگر حداقل یک درس با نوع نمره خام ثبت شده باشد → نمره‌ای
  if (rows.some((r) => r.scoreType === "raw")) return false;
  return true;
}

function examQuestionStats(ex, subjectId) {
  const n = Number(ex?.questionCount) || 0;
  const answers = ex?.answers || {};
  const qMeta = ex?.qMeta || {};
  let correct = 0, wrong = 0, blank = 0, total = 0;
  for (let i = 1; i <= n; i++) {
    if (subjectId) {
      const sid = qMeta[i]?.subjectId || qMeta[String(i)]?.subjectId;
      if (sid && sid !== subjectId) continue;
      if (!sid && subjectId) {
        // اگر مبحث ست نشده، فقط وقتی تک‌درسی است حساب کن
        const sids = (ex.subjects || []).map((s) => s.subjectId);
        if (sids.length !== 1 || sids[0] !== subjectId) continue;
      }
    }
    total++;
    const st = answers[i] ?? answers[String(i)];
    if (st === "correct") correct++;
    else if (st === "wrong") wrong++;
    else if (st === "blank") blank++;
  }
  return { correct, wrong, blank, total };
}



// ---------- Jalali (Persian) calendar — precise bidirectional conversion ----------
function jDiv(a, b) { return Math.trunc(a / b); }
function jMod(a, b) { return a - Math.trunc(a / b) * b; }
const JAL_BREAKS = [-61, 9, 38, 199, 426, 686, 756, 818, 1111, 1181, 1210, 1635, 2060, 2097, 2192, 2262, 2324, 2394, 2456, 3178];
function jalCal(jy) {
  const bl = JAL_BREAKS.length;
  let gy = jy + 621, leapJ = -14, jp = JAL_BREAKS[0], jm, jump, leap, n, i, leapG, march;
  for (i = 1; i < bl; i += 1) {
    jm = JAL_BREAKS[i]; jump = jm - jp;
    if (jy < jm) break;
    leapJ = leapJ + jDiv(jump, 33) * 8 + jDiv(jMod(jump, 33), 4);
    jp = jm;
  }
  n = jy - jp;
  leapJ = leapJ + jDiv(n, 33) * 8 + jDiv(jMod(n, 33) + 3, 4);
  if (jMod(jump, 33) === 4 && jump - n === 4) leapJ += 1;
  leapG = jDiv(gy, 4) - jDiv((jDiv(gy, 100) + 1) * 3, 4) - 150;
  march = 20 + leapJ - leapG;
  if (jump - n < 6) n = n - jump + jDiv(jump + 4, 33) * 33;
  leap = jMod(jMod(n + 1, 33) - 1, 4);
  if (leap === -1) leap = 4;
  return { leap, gy, march };
}
function g2d(gy, gm, gd) {
  let d = jDiv((gy + jDiv(gm - 8, 6) + 100100) * 1461, 4) + jDiv(153 * jMod(gm + 9, 12) + 2, 5) + gd - 34840408;
  d = d - jDiv(jDiv(gy + 100100 + jDiv(gm - 8, 6), 100) * 3, 4) + 752;
  return d;
}
function d2g(jdn) {
  let j = 4 * jdn + 139361631;
  j = j + jDiv(jDiv(4 * jdn + 183187720, 146097) * 3, 4) * 4 - 3908;
  const i = jDiv(jMod(j, 1461), 4) * 5 + 308;
  const gd = jDiv(jMod(i, 153), 5) + 1;
  const gm = jMod(jDiv(i, 153), 12) + 1;
  const gy = jDiv(j, 1461) - 100100 + jDiv(8 - gm, 6);
  return { gy, gm, gd };
}
function j2d(jy, jm, jd) {
  const r = jalCal(jy);
  return g2d(r.gy, 3, r.march) + (jm - 1) * 31 - jDiv(jm, 7) * (jm - 7) + jd - 1;
}
function d2j(jdn) {
  const gy = d2g(jdn).gy;
  let jy = gy - 621;
  const r = jalCal(jy);
  const jdn1f = g2d(gy, 3, r.march);
  let k = jdn - jdn1f, jd, jm;
  if (k >= 0) {
    if (k <= 185) { jm = 1 + jDiv(k, 31); jd = jMod(k, 31) + 1; return { jy, jm, jd }; }
    k -= 186;
  } else { jy -= 1; k += 179; if (r.leap === 1) k += 1; }
  jm = 7 + jDiv(k, 30); jd = jMod(k, 30) + 1;
  return { jy, jm, jd };
}
function gregorianToJalaliYMD(gy, gm, gd) { return d2j(g2d(gy, gm, gd)); }
function jalaliToGregorianISO(jy, jm, jd) {
  const g = d2g(j2d(jy, jm, jd));
  return `${g.gy}-${String(g.gm).padStart(2, "0")}-${String(g.gd).padStart(2, "0")}`;
}
function daysInJalaliMonth(jy, jm) {
  if (jm <= 6) return 31;
  if (jm <= 11) return 30;
  return j2d(jy + 1, 1, 1) - j2d(jy, 1, 1) - 336;
}
function toJalaliStr(date) {
  const { jy, jm, jd } = gregorianToJalaliYMD(date.getFullYear(), date.getMonth() + 1, date.getDate());
  return `${toFa(jd)} ${JALALI_MONTHS[jm - 1]} ${toFa(jy)}`;
}
function toJalaliShort(date) {
  const { jm, jd } = gregorianToJalaliYMD(date.getFullYear(), date.getMonth() + 1, date.getDate());
  return `${toFa(jd)} ${JALALI_MONTHS[jm - 1]}`;
}
function getCurrentWeekDates() {
  const today = new Date();
  const iranianIdx = (today.getDay() + 1) % 7; // 0=شنبه ... 6=جمعه
  const start = new Date(today);
  start.setDate(today.getDate() - iranianIdx);
  return Array.from({ length: 7 }, (_, i) => {
    const d = new Date(start);
    d.setDate(start.getDate() + i);
    return d;
  });
}
function currentJalaliMonthKeys() {
  const now = new Date();
  const { jy, jm } = gregorianToJalaliYMD(now.getFullYear(), now.getMonth() + 1, now.getDate());
  const days = daysInJalaliMonth(jy, jm);
  return Array.from({ length: days }, (_, i) => jalaliToGregorianISO(jy, jm, i + 1));
}

// ---------- helpers ----------
const FA_DIGITS = ["۰", "۱", "۲", "۳", "۴", "۵", "۶", "۷", "۸", "۹"];
const toFa = (n) => String(n).replace(/[0-9]/g, (d) => FA_DIGITS[d]);

function formatDuration(mins) {
  mins = Math.round(mins);
  const h = Math.floor(mins / 60);
  const m = mins % 60;
  if (h === 0) return `${toFa(m)} دقیقه`;
  if (m === 0) return `${toFa(h)} ساعت`;
  return `${toFa(h)} ساعت و ${toFa(m)} دقیقه`;
}

function todayKey(d = new Date()) {
  return d.toISOString().slice(0, 10);
}

function lastNDays(n) {
  const out = [];
  for (let i = n - 1; i >= 0; i--) {
    const d = new Date();
    d.setDate(d.getDate() - i);
    out.push(todayKey(d));
  }
  return out;
}

const WEEKDAY_FA = ["ی", "د", "س", "چ", "پ", "ج", "ش"];
function dayLabel(dateStr) {
  const d = new Date(dateStr + "T00:00:00");
  return WEEKDAY_FA[d.getDay()];
}

function computeStreak(sessions) {
  const daysWithSession = new Set(sessions.map((s) => s.date));
  let streak = 0;
  let cursor = new Date();
  while (true) {
    const key = todayKey(cursor);
    if (daysWithSession.has(key)) {
      streak += 1;
      cursor.setDate(cursor.getDate() - 1);
    } else {
      if (streak === 0 && key === todayKey()) {
        cursor.setDate(cursor.getDate() - 1);
        continue;
      }
      break;
    }
  }
  return streak;
}

// aggregate a user's sessions into per-day totals (minutes / tests / exercises)
// used both locally and as the payload published to the shared leaderboard
function buildDailyStats(sessions) {
  const map = {};
  sessions.forEach((s) => {
    if (!map[s.date]) map[s.date] = { minutes: 0, tests: 0, exercises: 0 };
    map[s.date].minutes += s.minutes;
    if (s.exerciseCount) {
      if (s.exerciseType === "تست") map[s.date].tests += s.exerciseCount;
      else map[s.date].exercises += s.exerciseCount;
    }
  });
  return map;
}

// ---------- storage ----------
function uid() { return Date.now().toString(36) + Math.random().toString(36).slice(2, 8); }

function loadKey(key, fallback, shared = false) {
  try {
    const raw = localStorage.getItem(key);
    return raw ? JSON.parse(raw) : fallback;
  } catch {
    return fallback;
  }
}
function saveKey(key, value, shared = false) {
  try {
    localStorage.setItem(key, JSON.stringify(value));
  } catch {
    /* ignore */
  }
}
function listKeys(prefix, shared = false) {
  try {
    const keys = [];
    for (let i = 0; i < localStorage.length; i++) {
      const k = localStorage.key(i);
      if (k && k.startsWith(prefix)) keys.push(k);
    }
    return keys;
  } catch {
    return [];
  }
}

const LEADERBOARD_PREFIX = "kanoon:leaderboard:";

// ---------- shared: grouped curriculum picker ----------
function GroupedSubjectPicker({ items, mode, value, onChange }) {
  const groups = GROUP_ORDER
    .map((g) => ({ label: g, items: items.filter((i) => i.groupLabel === g) }))
    .filter((g) => g.items.length > 0);
  const isSelected = (id) => (mode === "multi" ? value.includes(id) : value === id);
  const toggle = (id) => {
    if (mode === "multi") {
      onChange(value.includes(id) ? value.filter((x) => x !== id) : [...value, id]);
    } else {
      onChange(id);
    }
  };
  return (
    <div>
      {groups.map((g) => (
        <div key={g.label} style={{ marginBottom: 10 }}>
          <div style={{ fontSize: 10.5, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>{g.label}</div>
          <div style={{ display: "flex", gap: 6, flexWrap: "wrap" }}>
            {g.items.map((it) => (
              <button key={it.id} onClick={() => toggle(it.id)} className="kn-btn" style={{
                border: `1.5px solid ${isSelected(it.id) ? it.color : COLORS.border}`,
                background: isSelected(it.id) ? it.color + "22" : "transparent",
                color: isSelected(it.id) ? COLORS.text : COLORS.muted,
                borderRadius: 999, padding: "5px 11px", fontSize: 12, fontWeight: 600,
              }}>{it.name}</button>
            ))}
          </div>
        </div>
      ))}
    </div>
  );
}

// ---------- shared: Jalali date picker (day/month/year selects) ----------
function JalaliDatePicker({ value, onChange }) {
  const d = new Date(value + "T00:00:00");
  const cur = gregorianToJalaliYMD(d.getFullYear(), d.getMonth() + 1, d.getDate());
  const dim = daysInJalaliMonth(cur.jy, cur.jm);
  const nowJ = gregorianToJalaliYMD(new Date().getFullYear(), new Date().getMonth() + 1, new Date().getDate());
  const years = [];
  for (let y = nowJ.jy - 3; y <= nowJ.jy; y++) years.push(y);

  function set(jy, jm, jd) {
    const clampedDay = Math.min(jd, daysInJalaliMonth(jy, jm));
    onChange(jalaliToGregorianISO(jy, jm, clampedDay));
  }
  const selStyle = {
    background: COLORS.bg, border: `1px solid ${COLORS.border}`, borderRadius: 8,
    padding: "7px 4px", color: COLORS.text, fontSize: 11.5, outline: "none",
  };
  return (
    <div style={{ display: "flex", gap: 6 }}>
      <select value={cur.jd} onChange={(e) => set(cur.jy, cur.jm, Number(e.target.value))} style={{ ...selStyle, flex: 0.75 }}>
        {Array.from({ length: dim }, (_, i) => i + 1).map((n) => <option key={n} value={n}>{toFa(n)}</option>)}
      </select>
      <select value={cur.jm} onChange={(e) => set(cur.jy, Number(e.target.value), cur.jd)} style={{ ...selStyle, flex: 1.6 }}>
        {JALALI_MONTHS.map((m, i) => <option key={m} value={i + 1}>{m}</option>)}
      </select>
      <select value={cur.jy} onChange={(e) => set(Number(e.target.value), cur.jm, cur.jd)} style={{ ...selStyle, flex: 1 }}>
        {years.map((y) => <option key={y} value={y}>{toFa(y)}</option>)}
      </select>
    </div>
  );
}

// ---------- profile onboarding / edit ----------
const GRADE_OPTIONS = ["دهم", "یازدهم", "دوازدهم"];

function AuthView({ onSuccess }) {
  const [mode, setMode] = useState("login"); // login | register
  const [username, setUsername] = useState("");
  const [password, setPassword] = useState("");
  const [showPass, setShowPass] = useState(false);
  const [name, setName] = useState("");
  const [grade, setGrade] = useState("دوازدهم");
  const [error, setError] = useState("");
  const [busy, setBusy] = useState(false);

  async function submit() {
    setError("");
    setBusy(true);
    try {
      let profile;
      if (mode === "login") {
        profile = await loginUser(username, password);
      } else {
        if (!name.trim()) throw new Error("نام نمایشی را وارد کن");
        profile = await registerUser(username, password, name, grade);
      }
      onSuccess(profile);
    } catch (e) {
      console.error(e);
      let msg = e.message || "خطا در ورود";
      if (e.code === "auth/email-already-in-use") msg = "این یوزرنیم قبلاً ثبت شده";
      if (e.code === "auth/wrong-password" || e.code === "auth/invalid-credential") msg = "یوزرنیم یا رمز اشتباه است";
      if (e.code === "auth/user-not-found") msg = "کاربری با این یوزرنیم پیدا نشد";
      if (e.code === "auth/weak-password") msg = "رمز خیلی ضعیف است (حداقل ۶ حرف)";
      if (e.code === "auth/operation-not-allowed") msg = "ورود با ایمیل در Firebase فعال نیست. Email/Password را روشن کن.";
      setError(msg);
    }
    setBusy(false);
  }

  const inputStyle = {
    width: "100%", background: COLORS.surface2, border: `1px solid ${COLORS.border}`,
    borderRadius: 10, padding: "10px 12px", color: COLORS.text, fontSize: 14, outline: "none", marginBottom: 12,
  };

  return (
    <div dir="rtl" style={{
      fontFamily: "'Vazirmatn', Tahoma, sans-serif", background: COLORS.bg, color: COLORS.text,
      minHeight: "100vh", display: "flex", alignItems: "center", justifyContent: "center", padding: 20,
    }}>
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Vazirmatn:wght@400;500;600;700;800&display=swap');
        *, button, input { font-family: 'Vazirmatn', Tahoma, sans-serif; }
        * { box-sizing: border-box; }
        button { cursor: pointer; }
      `}</style>
      <div style={{ width: "100%", maxWidth: 360, background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 18, padding: 22 }}>
        <div style={{ fontSize: 22, fontWeight: 800, textAlign: "center", marginBottom: 4 }}>کانون</div>
        <div style={{ fontSize: 12.5, color: COLORS.muted, textAlign: "center", marginBottom: 18 }}>
          {mode === "login" ? "ورود به حساب" : "ساخت حساب جدید"}
        </div>

        <div style={{ display: "flex", gap: 8, marginBottom: 16 }}>
          <button type="button" onClick={() => { setMode("login"); setError(""); }} className="kn-btn" style={{
            flex: 1, border: `1.5px solid ${mode === "login" ? COLORS.accent : COLORS.border}`,
            background: mode === "login" ? COLORS.accent + "22" : "transparent",
            color: mode === "login" ? COLORS.accentSoft : COLORS.muted,
            borderRadius: 10, padding: "8px 0", fontSize: 13, fontWeight: 700,
          }}>ورود</button>
          <button type="button" onClick={() => { setMode("register"); setError(""); }} className="kn-btn" style={{
            flex: 1, border: `1.5px solid ${mode === "register" ? COLORS.accent : COLORS.border}`,
            background: mode === "register" ? COLORS.accent + "22" : "transparent",
            color: mode === "register" ? COLORS.accentSoft : COLORS.muted,
            borderRadius: 10, padding: "8px 0", fontSize: 13, fontWeight: 700,
          }}>ثبت‌نام</button>
        </div>

        <div style={{ fontSize: 11.5, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>یوزرنیم</div>
        <input value={username} onChange={(e) => setUsername(e.target.value)} placeholder="مثلاً ali_m" autoFocus
          autoCapitalize="none" autoCorrect="off" style={inputStyle} />

        <div style={{ fontSize: 11.5, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>رمز عبور</div>
        <div style={{ position: "relative", marginBottom: 12 }}>
          <input
            type={showPass ? "text" : "password"}
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            placeholder="حداقل ۶ حرف"
            onKeyDown={(e) => e.key === "Enter" && submit()}
            style={{ ...inputStyle, marginBottom: 0, paddingLeft: 44 }}
          />
          <button
            type="button"
            onClick={() => setShowPass((v) => !v)}
            className="kn-btn"
            style={{
              position: "absolute", left: 8, top: "50%", transform: "translateY(-50%)",
              background: "none", border: "none", color: COLORS.muted, fontSize: 12, fontWeight: 700, padding: "4px 8px",
            }}
          >{showPass ? "مخفی" : "نمایش"}</button>
        </div>

        {mode === "register" && (
          <>
            <div style={{ fontSize: 11.5, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>نام نمایشی</div>
            <input value={name} onChange={(e) => setName(e.target.value)} placeholder="مثلاً علی محمدی" style={inputStyle} />
            <div style={{ fontSize: 11.5, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>پایه</div>
            <div style={{ display: "flex", gap: 8, marginBottom: 14 }}>
              {GRADE_OPTIONS.map((g) => (
                <button key={g} type="button" onClick={() => setGrade(g)} className="kn-btn" style={{
                  flex: 1, border: `1.5px solid ${grade === g ? COLORS.accent : COLORS.border}`,
                  background: grade === g ? COLORS.accent + "22" : "transparent",
                  color: grade === g ? COLORS.accentSoft : COLORS.muted,
                  borderRadius: 10, padding: "8px 0", fontSize: 12.5, fontWeight: 700,
                }}>{g}</button>
              ))}
            </div>
          </>
        )}

        {error && (
          <div style={{ fontSize: 12, color: COLORS.danger, background: COLORS.danger + "18", borderRadius: 10, padding: "8px 12px", marginBottom: 12 }}>
            {error}
          </div>
        )}

        <button onClick={submit} disabled={busy || !username.trim() || !password} className="kn-btn" style={{
          width: "100%", background: (!username.trim() || !password || busy) ? COLORS.surface2 : COLORS.accent,
          color: (!username.trim() || !password || busy) ? COLORS.muted : "#1a1400",
          border: "none", borderRadius: 10, padding: "11px 0", fontSize: 14, fontWeight: 700,
        }}>{busy ? "لطفاً صبر کن..." : (mode === "login" ? "ورود" : "ثبت‌نام")}</button>
      </div>
    </div>
  );
}

function ProfileEditView({ initial, onSave, onCancel, onLogout, settings, setSettings }) {
  const [name, setName] = useState(initial?.name ?? "");
  const [grade, setGrade] = useState(initial?.grade ?? "دوازدهم");
  const hidePlan = !!(settings && settings.hidePlanTab);

  async function submit() {
    if (!name.trim()) return;
    const updated = { ...initial, name: name.trim(), grade };
    try {
      await db.collection(USERS_COLLECTION).doc(initial.userId).set(updated, { merge: true });
    } catch (e) { console.error(e); }
    onSave(updated);
  }

  function togglePlanTab() {
    if (setSettings) {
      setSettings((prev) => ({ ...prev, hidePlanTab: !prev.hidePlanTab }));
    }
  }

  return (
    <div dir="rtl" style={{
      fontFamily: "'Vazirmatn', Tahoma, sans-serif", background: COLORS.bg, color: COLORS.text,
      minHeight: "100vh", display: "flex", alignItems: "center", justifyContent: "center", padding: 20,
    }}>
      <div style={{ width: "100%", maxWidth: 340, background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 18, padding: 22 }}>
        <div style={{ fontSize: 18, fontWeight: 800, textAlign: "center", marginBottom: 16 }}>ویرایش پروفایل</div>
        <div style={{ fontSize: 11.5, color: COLORS.muted, marginBottom: 12 }}>
          یوزرنیم: <b style={{ color: COLORS.text }}>{initial?.username || "—"}</b>
        </div>
        <div style={{ fontSize: 11.5, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>نام نمایشی</div>
        <input value={name} onChange={(e) => setName(e.target.value)}
          style={{ width: "100%", background: COLORS.surface2, border: `1px solid ${COLORS.border}`, borderRadius: 10, padding: "10px 12px", color: COLORS.text, fontSize: 14, outline: "none", marginBottom: 14 }} />
        <div style={{ fontSize: 11.5, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>پایه</div>
        <div style={{ display: "flex", gap: 8, marginBottom: 18 }}>
          {GRADE_OPTIONS.map((g) => (
            <button key={g} type="button" onClick={() => setGrade(g)} className="kn-btn" style={{
              flex: 1, border: `1.5px solid ${grade === g ? COLORS.accent : COLORS.border}`,
              background: grade === g ? COLORS.accent + "22" : "transparent",
              color: grade === g ? COLORS.accentSoft : COLORS.muted,
              borderRadius: 10, padding: "8px 0", fontSize: 12.5, fontWeight: 700,
            }}>{g}</button>
          ))}
        </div>
        <button onClick={submit} className="kn-btn" style={{
          width: "100%", background: COLORS.accent, color: "#1a1400", border: "none", borderRadius: 10, padding: "11px 0", fontSize: 14, fontWeight: 700,
        }}>ذخیره</button>
        <button onClick={onCancel} className="kn-btn" style={{ width: "100%", background: "none", color: COLORS.muted, border: "none", padding: "10px 0", fontSize: 12.5, marginTop: 4 }}>انصراف</button>
        <button onClick={onLogout} className="kn-btn" style={{
          width: "100%", background: COLORS.danger + "22", color: COLORS.danger, border: "none", borderRadius: 10, padding: "10px 0", fontSize: 13, fontWeight: 700, marginTop: 8,
        }}>خروج از حساب</button>
        <div style={{ marginTop: 18, paddingTop: 12, borderTop: `1px solid ${COLORS.border}` }}>
          <button type="button" onClick={togglePlanTab} className="kn-btn" style={{
            width: "100%", background: "none", border: "none", color: COLORS.muted, fontSize: 11, padding: "6px 0", textAlign: "center",
          }}>
            {hidePlan ? "نمایش دوباره تب برنامه در نوار پایین" : "پنهان کردن تب برنامه از نوار پایین"}
          </button>
        </div>
      </div>
    </div>
  );
}

function AdminPanel({ profile, onBack }) {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [msg, setMsg] = useState("");
  const [busy, setBusy] = useState(false);

  async function refresh() {
    setLoading(true);
    try {
      const list = await listAllUsers();
      setUsers(list.filter((u) => u.role !== "admin" && u.username !== "admin"));
    } catch (e) {
      console.error(e);
      setMsg("خطا در دریافت لیست کاربران");
    }
    setLoading(false);
  }

  useEffect(() => { refresh(); }, []);

  async function handleDelete(u) {
    if (!confirm(`کاربر «${u.name || u.username}» حذف شود؟ از رقابت و لیست کاربران پاک می‌شود.`)) return;
    try {
      await deleteUserAsAdmin(u.userId);
      setMsg(`«${u.name || u.username}» حذف شد`);
      refresh();
    } catch (e) {
      console.error(e);
      setMsg("حذف ناموفق بود");
    }
  }

  async function handleResetPassword(u) {
    const np = prompt(`رمز جدید برای «${u.name || u.username}» (@${u.username}):\nحداقل ۶ حرف`);
    if (np === null) return;
    if (!np || np.length < 6) {
      setMsg("رمز باید حداقل ۶ حرف باشد");
      return;
    }
    try {
      await adminResetPassword(u.userId, np);
      setMsg(`رمز «${u.name || u.username}» تغییر کرد. رمز جدید را به او بگو.`);
    } catch (e) {
      console.error(e);
      setMsg(e.message || "تغییر رمز ناموفق بود");
    }
  }

  async function handleResetLeaderboard() {
    if (!confirm("لیست رقابت کاملاً ریست شود؟ ساعت مطالعه و تست‌های همه از جدول رقابت پاک می‌شود (داده محلی هر نفر روی گوشی‌اش می‌ماند).")) return;
    setBusy(true);
    try {
      const n = await resetLeaderboard();
      setMsg(n === 0 ? "لیست رقابت خالی بود" : `لیست رقابت ریست شد (${n} مورد حذف شد)`);
    } catch (e) {
      console.error(e);
      setMsg("ریست رقابت ناموفق بود");
    }
    setBusy(false);
  }

  return (
    <div className="kn-fade">
      <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 14 }}>
        <div style={{ fontSize: 15, fontWeight: 700 }}>پنل ادمین</div>
        <button onClick={onBack} className="kn-btn" style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 10, padding: "6px 12px", color: COLORS.text, fontSize: 12 }}>بازگشت</button>
      </div>

      <div style={{
        background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14,
        padding: 14, marginBottom: 16,
      }}>
        <div style={{ fontSize: 13, fontWeight: 700, marginBottom: 6 }}>رقابت</div>
        <div style={{ fontSize: 11.5, color: COLORS.muted, marginBottom: 12, lineHeight: 1.7 }}>
          با ریست کردن، تمام رکوردهای جدول رقابت از سرور پاک می‌شود. حساب کاربران حذف نمی‌شود.
        </div>
        <button onClick={handleResetLeaderboard} disabled={busy} className="kn-btn" style={{
          width: "100%", background: COLORS.danger + "22", color: COLORS.danger, border: `1px solid ${COLORS.danger}55`,
          borderRadius: 10, padding: "10px 0", fontSize: 13, fontWeight: 700,
        }}>{busy ? "در حال ریست..." : "ریست لیست رقابت"}</button>
      </div>

      <div style={{ fontSize: 13, fontWeight: 700, marginBottom: 10, color: COLORS.muted }}>کاربران</div>
      {msg && <div style={{ fontSize: 12, color: COLORS.accentSoft, marginBottom: 10 }}>{msg}</div>}
      {loading ? (
        <div style={{ color: COLORS.muted, textAlign: "center", padding: 40 }}>در حال بارگذاری...</div>
      ) : users.length === 0 ? (
        <div style={{ color: COLORS.muted, textAlign: "center", padding: 40 }}>کاربری ثبت نشده</div>
      ) : (
        <div style={{ display: "flex", flexDirection: "column", gap: 8 }}>
          {users.map((u) => (
            <div key={u.userId} style={{
              background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 12,
              padding: "12px 14px", display: "flex", alignItems: "center", justifyContent: "space-between",
            }}>
              <div>
                <div style={{ fontSize: 13, fontWeight: 700 }}>{u.name}</div>
                <div style={{ fontSize: 11, color: COLORS.muted }}>@{u.username} · {u.grade}</div>
              </div>
              <div style={{ display: "flex", gap: 6 }}>
                <button onClick={() => handleResetPassword(u)} className="kn-btn" style={{
                  background: COLORS.accent + "22", color: COLORS.accentSoft, border: "none", borderRadius: 8, padding: "6px 10px", fontSize: 11.5, fontWeight: 700,
                }}>رمز جدید</button>
                <button onClick={() => handleDelete(u)} className="kn-btn" style={{
                  background: COLORS.danger + "22", color: COLORS.danger, border: "none", borderRadius: 8, padding: "6px 10px", fontSize: 11.5, fontWeight: 700,
                }}>حذف</button>
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}

function LoadingScreen() {
  return (
    <div dir="rtl" style={{
      fontFamily: "'Vazirmatn', Tahoma, sans-serif", background: COLORS.bg, color: COLORS.muted,
      minHeight: "100vh", display: "flex", alignItems: "center", justifyContent: "center",
    }}>
      <style>{`@import url('https://fonts.googleapis.com/css2?family=Vazirmatn:wght@400;500;600;700;800&display=swap');`}</style>
      در حال بارگذاری...
    </div>
  );
}

// ================= APP =================
function userStorageKey(userId, name) {
  return `kanoon:u:${userId}:${name}`;
}

function loadUserData(userId) {
  if (!userId) {
    return {
      boxes: [],
      sessions: [],
      tasks: [],
      settings: { normalHours: DEFAULT_NORMAL_DAY_HOURS, holidayHours: DEFAULT_HOLIDAY_DAY_HOURS },
      dayOverrides: {},
      weeklyPlan: { title: "برنامه مطالعاتی هفتگی", items: [] },
      exams: [],
    };
  }
  return {
    boxes: loadKey(userStorageKey(userId, "boxes"), []),
    sessions: loadKey(userStorageKey(userId, "sessions"), []),
    tasks: loadKey(userStorageKey(userId, "tasks"), []),
    settings: loadKey(userStorageKey(userId, "settings"), { normalHours: DEFAULT_NORMAL_DAY_HOURS, holidayHours: DEFAULT_HOLIDAY_DAY_HOURS }),
    dayOverrides: loadKey(userStorageKey(userId, "dayOverrides"), {}),
    weeklyPlan: loadKey(userStorageKey(userId, "weeklyPlan"), { title: "برنامه مطالعاتی هفتگی", items: [] }),
    exams: loadKey(userStorageKey(userId, "exams"), []),
  };
}

function App() {
  const [loading, setLoading] = useState(true);
  const [profile, setProfile] = useState(null);
  const [editingProfile, setEditingProfile] = useState(false);
  const [boxes, setBoxes] = useState([]);
  const [sessions, setSessions] = useState([]);
  const [tasks, setTasks] = useState([]);
  const [settings, setSettings] = useState({ normalHours: DEFAULT_NORMAL_DAY_HOURS, holidayHours: DEFAULT_HOLIDAY_DAY_HOURS });
  const [dayOverrides, setDayOverrides] = useState({});
  const [weeklyPlan, setWeeklyPlan] = useState({ title: "برنامه مطالعاتی هفتگی", items: [] });
  const [exams, setExams] = useState([]);
  const [tab, setTab] = useState("boxes");
  const loadedRef = useRef(false);
  const currentUserIdRef = useRef(null);

  function applyUserData(userId, override) {
    const local = loadUserData(userId);
    currentUserIdRef.current = userId || null;
    const boxes = override?.boxes != null ? override.boxes : local.boxes;
    const sessions = override?.sessions != null ? override.sessions : local.sessions;
    const settings = override?.settings != null ? override.settings : local.settings;
    const dayOverrides = override?.dayOverrides != null ? override.dayOverrides : local.dayOverrides;
    const weeklyPlan = override?.weeklyPlan != null ? override.weeklyPlan : local.weeklyPlan;
    const exams = override?.exams != null ? override.exams : local.exams;
    setBoxes(boxes);
    setSessions(sessions);
    setTasks(local.tasks); // تکالیف مشترک جداگانه از Firebase می‌آید
    setSettings(settings);
    setDayOverrides(dayOverrides);
    setWeeklyPlan(weeklyPlan || { title: "برنامه مطالعاتی هفتگی", items: [] });
    setExams(exams || []);
    // ذخیره محلی پس از merge
    if (userId) {
      saveKey(userStorageKey(userId, "boxes"), boxes);
      saveKey(userStorageKey(userId, "sessions"), sessions);
      saveKey(userStorageKey(userId, "settings"), settings);
      saveKey(userStorageKey(userId, "dayOverrides"), dayOverrides);
      saveKey(userStorageKey(userId, "weeklyPlan"), weeklyPlan || { title: "برنامه مطالعاتی هفتگی", items: [] });
      saveKey(userStorageKey(userId, "exams"), exams || []);
    }
  }

  async function loadUserWithCloud(p) {
    if (!p?.userId) {
      applyUserData(null);
      return;
    }
    const local = loadUserData(p.userId);
    const remote = await pullUserStudy(p.userId);
    if (remote) {
      const sessions = mergeSessions(local.sessions, remote.sessions || []);
      const boxes = mergeBoxes(local.boxes, remote.boxes || []);
      const settings = remote.settings || local.settings;
      const dayOverrides = { ...(local.dayOverrides || {}), ...(remote.dayOverrides || {}) };
      const weeklyPlan = remote.weeklyPlan || local.weeklyPlan;
      const exams = remote.exams || local.exams || [];
      applyUserData(p.userId, { sessions, boxes, settings, dayOverrides, weeklyPlan, exams });
    } else {
      applyUserData(p.userId);
      // اگر ابر خالی است ولی محلی داده دارد، آپلود کن
      if ((local.sessions && local.sessions.length) || (local.boxes && local.boxes.length)) {
        await pushUserStudy(p.userId, local);
      }
    }
  }

  function handleLoginSuccess(p) {
    loadedRef.current = false;
    setProfile(p);
    (async () => {
      try {
        await loadUserWithCloud(p);
      } catch (e) {
        console.error(e);
        applyUserData(p?.userId);
      }
      setTimeout(() => { loadedRef.current = true; }, 50);
    })();
  }

  useEffect(() => {
    (async () => {
      try {
        const p = await restoreSession();
        if (p) {
          setProfile(p);
          await loadUserWithCloud(p);
        } else {
          setProfile(null);
          applyUserData(null);
        }
      } catch (e) {
        console.error(e);
        setProfile(null);
        applyUserData(null);
      }
      setLoading(false);
      loadedRef.current = true;
    })();
  }, []);

  // ذخیره محلی فقط برای همان کاربر فعلی
  useEffect(() => {
    if (!loadedRef.current || !currentUserIdRef.current) return;
    saveKey(userStorageKey(currentUserIdRef.current, "boxes"), boxes);
  }, [boxes]);
  useEffect(() => {
    if (!loadedRef.current || !currentUserIdRef.current) return;
    saveKey(userStorageKey(currentUserIdRef.current, "sessions"), sessions);
  }, [sessions]);
  useEffect(() => {
    if (!loadedRef.current || !currentUserIdRef.current) return;
    saveKey(userStorageKey(currentUserIdRef.current, "tasks"), tasks);
  }, [tasks]);
  useEffect(() => {
    if (!loadedRef.current || !currentUserIdRef.current) return;
    saveKey(userStorageKey(currentUserIdRef.current, "settings"), settings);
  }, [settings]);
  useEffect(() => {
    if (!loadedRef.current || !currentUserIdRef.current) return;
    saveKey(userStorageKey(currentUserIdRef.current, "dayOverrides"), dayOverrides);
  }, [dayOverrides]);
  useEffect(() => {
    if (!loadedRef.current || !currentUserIdRef.current) return;
    saveKey(userStorageKey(currentUserIdRef.current, "weeklyPlan"), weeklyPlan);
  }, [weeklyPlan]);
  useEffect(() => {
    if (!loadedRef.current || !currentUserIdRef.current) return;
    saveKey(userStorageKey(currentUserIdRef.current, "exams"), exams);
  }, [exams]);

  // همگام‌سازی مطالعه با Firebase (بین دستگاه‌ها)
  useEffect(() => {
    if (!loadedRef.current || !profile || !profile.userId) return;
    if (profile.role === "admin") return;
    const t = setTimeout(() => {
      pushUserStudy(profile.userId, { sessions, boxes, settings, dayOverrides, weeklyPlan, exams });
      publishToLeaderboard(profile, sessions);
    }, 400);
    return () => clearTimeout(t);
  }, [sessions, boxes, settings, dayOverrides, weeklyPlan, exams, profile]);

  const streak = useMemo(() => computeStreak(sessions), [sessions]);

  useEffect(() => {
    if (settings?.hidePlanTab && tab === "plan") setTab("boxes");
  }, [settings?.hidePlanTab, tab]);

  if (loading) return <LoadingScreen />;

  if (!profile) {
    return <AuthView onSuccess={handleLoginSuccess} />;
  }

  if (editingProfile) {
    return (
      <ProfileEditView
        initial={profile}
        settings={settings}
        setSettings={setSettings}
        onSave={(p) => { setProfile(p); setEditingProfile(false); }}
        onCancel={() => setEditingProfile(false)}
        onLogout={async () => {
          await logoutUser();
          loadedRef.current = false;
          setProfile(null);
          applyUserData(null);
          setEditingProfile(false);
          setTab("boxes");
          setTimeout(() => { loadedRef.current = true; }, 0);
        }}
      />
    );
  }

  return (
    <div dir="rtl" style={{
      fontFamily: "'Vazirmatn', Tahoma, sans-serif",
      background: COLORS.bg, color: COLORS.text,
      minHeight: "100vh", display: "flex", flexDirection: "column",
    }}>
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Vazirmatn:wght@400;500;600;700;800&display=swap');
        *, button, input, select, textarea { font-family: 'Vazirmatn', Tahoma, sans-serif; }
        svg text, .recharts-wrapper, .recharts-text, .recharts-tooltip-wrapper { font-family: 'Vazirmatn', Tahoma, sans-serif !important; }
        * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
        body { margin: 0; }
        button { cursor: pointer; }
        ::-webkit-scrollbar { width: 6px; height: 6px; }
        ::-webkit-scrollbar-thumb { background: ${COLORS.border}; border-radius: 4px; }
        .kn-btn { transition: transform .12s ease, opacity .12s ease; }
        .kn-btn:active { transform: scale(0.96); }
        .kn-btn:disabled { opacity: .55; }
        .kn-tab { transition: color .15s ease; }
        .kn-fade { animation: kn-fade-in .25s ease; }
        @keyframes kn-fade-in { from { opacity: 0; transform: translateY(4px);} to { opacity: 1; transform: translateY(0);} }
        @keyframes kn-spin { from { transform: rotate(0deg); } to { transform: rotate(360deg); } }
        .kn-spin { animation: kn-spin 0.9s linear infinite; }
        @media (prefers-reduced-motion: reduce) {
          .kn-btn, .kn-fade { animation: none !important; transition: none !important; }
        }
      `}</style>

      {/* header */}
      <header style={{
        padding: "18px 20px 14px", display: "flex", alignItems: "center",
        justifyContent: "space-between", borderBottom: `1px solid ${COLORS.border}`,
      }}>
        <div>
          <div style={{ fontSize: 22, fontWeight: 800, letterSpacing: "-0.02em" }}>کانون</div>
          <div style={{ fontSize: 12, color: COLORS.muted, marginTop: 2 }}>دستیار مطالعه و تحلیل عملکرد</div>
          <div style={{ fontSize: 11.5, color: COLORS.accentSoft, marginTop: 3, fontWeight: 600 }}>
            {WEEKDAY_FULL_FA[new Date().getDay()]}، {toJalaliStr(new Date())}
          </div>
        </div>
        <div style={{ display: "flex", flexDirection: "column", alignItems: "flex-end", gap: 6 }}>
          <div style={{
            display: "flex", alignItems: "center", gap: 6, background: COLORS.surface,
            padding: "8px 12px", borderRadius: 999, border: `1px solid ${COLORS.border}`,
          }}>
            <Flame size={16} color={streak > 0 ? COLORS.accent : COLORS.muted} />
            <span style={{ fontSize: 13, fontWeight: 700, color: streak > 0 ? COLORS.accent : COLORS.muted }}>
              {toFa(streak)} روز
            </span>
          </div>
          <button onClick={() => setEditingProfile(true)} className="kn-btn" style={{
            display: "flex", alignItems: "center", gap: 4, background: "none", border: "none",
            color: COLORS.muted, fontSize: 11, padding: "2px 4px",
          }}>
            <Pencil size={11} /> {profile.name} · {profile.grade}
          </button>
        </div>
      </header>

      {/* content */}
      <main style={{ flex: 1, overflowY: "auto", padding: "16px 16px 90px", maxWidth: 640, margin: "0 auto", width: "100%" }}>
        {tab === "boxes" ? (
          <BoxesView
            boxes={boxes} setBoxes={setBoxes}
            sessions={sessions} setSessions={setSessions}
            settings={settings} setSettings={setSettings}
            dayOverrides={dayOverrides} setDayOverrides={setDayOverrides}
          />
        ) : tab === "homework" ? (
          <HomeworkView profile={profile} />
        ) : tab === "plan" ? (
          <WeeklyPlanView weeklyPlan={weeklyPlan} setWeeklyPlan={setWeeklyPlan} profile={profile} />
        ) : tab === "exams" ? (
          <ExamsView exams={exams} setExams={setExams} sessions={sessions} profile={profile} />
        ) : tab === "competition" ? (
          <CompetitionView profile={profile} />
        ) : tab === "admin" ? (
          <AdminPanel profile={profile} onBack={() => setTab("boxes")} />
        ) : (
          <AnalyticsView
            sessions={sessions} boxes={boxes} streak={streak}
            settings={settings} dayOverrides={dayOverrides}
            profile={profile}
          />
        )}
      </main>

      {/* bottom nav */}
      <nav style={{
        position: "fixed", bottom: 0, left: 0, right: 0, background: COLORS.surface,
        borderTop: `1px solid ${COLORS.border}`, display: "flex", justifyContent: "space-around",
        padding: "10px 4px calc(10px + env(safe-area-inset-bottom))",
      }}>
        {[
          { id: "boxes", label: "باکس‌ها", icon: Box },
          { id: "homework", label: "تکالیف", icon: ClipboardList },
          ...(!settings?.hidePlanTab ? [{ id: "plan", label: "برنامه", icon: Calendar }] : []),
          { id: "exams", label: "آزمون", icon: FileText },
          { id: "competition", label: "رقابت", icon: Trophy },
          { id: "analytics", label: "تحلیل", icon: BarChart3 },
          ...(profile?.role === "admin" ? [{ id: "admin", label: "ادمین", icon: Users }] : []),
        ].map(({ id, label, icon: Icon }) => (
          <button key={id} onClick={() => setTab(id)} className="kn-tab" style={{
            background: "none", border: "none", display: "flex", flexDirection: "column",
            alignItems: "center", gap: 4, color: tab === id ? COLORS.accent : COLORS.muted,
            fontSize: 11.5, fontWeight: 600, padding: "4px 12px",
          }}>
            <Icon size={19} />
            {label}
          </button>
        ))}
      </nav>
    </div>
  );
}

// ================= BOXES =================
function BoxesView({ boxes, setBoxes, sessions, setSessions, settings, setSettings, dayOverrides, setDayOverrides }) {
  const [adding, setAdding] = useState(boxes.length === 0);
  const [selectedIds, setSelectedIds] = useState([]);
  const [goalEditOpen, setGoalEditOpen] = useState(false);
  const [goalMode, setGoalMode] = useState("today"); // "today" | "normal" | "holiday"
  const [goalHoursInput, setGoalHoursInput] = useState("");
  const [goalMinutesInput, setGoalMinutesInput] = useState("0");

  function createBox() {
    if (selectedIds.length === 0) return;
    const dayKey = todayKey();
    const number = boxes.filter((b) => b.dayKey === dayKey).length + 1;
    setBoxes((prev) => [...prev, { id: Date.now().toString(), dayKey, number, subjectIds: selectedIds, createdAt: Date.now() }]);
    setSelectedIds([]);
    setAdding(false);
  }
  function deleteBox(id) {
    setBoxes((prev) => prev.filter((b) => b.id !== id));
    setSessions((prev) => prev.filter((s) => s.boxId !== id));
  }
  function deleteSession(id) {
    setSessions((prev) => prev.filter((s) => s.id !== id));
  }

  const now = new Date();
  const holidayToday = isHoliday(now);
  const todayKeyStr = todayKey(now);
  const hasTodayOverride = dayOverrides[todayKeyStr] != null && dayOverrides[todayKeyStr] !== "";
  const targetTodayMin = dayTargetMinutes(now, settings, dayOverrides);
  const todaySessions = sessions.filter((s) => s.date === todayKeyStr).sort((a, b) => b.timestamp - a.timestamp);
  const studiedTodayMin = todaySessions.reduce((a, s) => a + s.minutes, 0);
  const todayPct = Math.min(100, Math.round((studiedTodayMin / targetTodayMin) * 100));
  // هر روز فقط باکس‌های همان روز نمایش داده می‌شوند (شماره از ۱)
  const todayBoxes = boxes.filter((b) => b.dayKey === todayKeyStr);
  const sortedBoxes = [...todayBoxes].sort((a, b) => b.createdAt - a.createdAt);

  function hoursToParts(totalHours) {
    const totalMin = Math.round(Number(totalHours) * 60);
    const h = Math.floor(totalMin / 60);
    const m = totalMin % 60;
    return { h, m };
  }

  function openGoalEdit(mode) {
    setGoalMode(mode);
    let totalH;
    if (mode === "today") {
      totalH = hasTodayOverride ? dayOverrides[todayKeyStr] : (targetTodayMin / 60);
    } else if (mode === "normal") {
      totalH = settings.normalHours ?? DEFAULT_NORMAL_DAY_HOURS;
    } else {
      totalH = settings.holidayHours ?? DEFAULT_HOLIDAY_DAY_HOURS;
    }
    const parts = hoursToParts(totalH);
    setGoalHoursInput(String(parts.h));
    setGoalMinutesInput(String(parts.m));
    setGoalEditOpen(true);
  }

  function saveGoal() {
    const h = parseInt(goalHoursInput || "0", 10);
    const m = parseInt(goalMinutesInput || "0", 10);
    if (isNaN(h) || h < 0 || h > 24) return;
    if (isNaN(m) || m < 0 || m > 59) return;
    const totalHours = h + (m / 60);
    if (totalHours <= 0 || totalHours > 24) return;
    if (goalMode === "today") {
      setDayOverrides((prev) => ({ ...prev, [todayKeyStr]: totalHours }));
    } else if (goalMode === "normal") {
      setSettings((prev) => ({ ...prev, normalHours: totalHours }));
    } else {
      setSettings((prev) => ({ ...prev, holidayHours: totalHours }));
    }
    setGoalEditOpen(false);
  }

  function clearTodayOverride() {
    setDayOverrides((prev) => {
      const next = { ...prev };
      delete next[todayKeyStr];
      return next;
    });
    setGoalEditOpen(false);
  }

  return (
    <div className="kn-fade">
      {/* today target */}
      <div style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14, padding: "13px 14px", marginBottom: 16 }}>
        <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 8 }}>
          <div style={{ display: "flex", alignItems: "center", gap: 8, flexWrap: "wrap" }}>
            <span style={{
              fontSize: 11, fontWeight: 700, padding: "3px 9px", borderRadius: 999,
              background: holidayToday ? COLORS.accent + "22" : COLORS.accent2 + "22",
              color: holidayToday ? COLORS.accent : COLORS.accent2,
            }}>{holidayToday ? "روز تعطیل" : "روز عادی"}</span>
            {hasTodayOverride && (
              <span style={{
                fontSize: 10, fontWeight: 700, padding: "2px 7px", borderRadius: 999,
                background: COLORS.accent + "33", color: COLORS.accentSoft,
              }}>استثنای امروز</span>
            )}
            <span style={{ fontSize: 12, color: COLORS.muted }}>هدف امروز: {formatDuration(targetTodayMin)}</span>
          </div>
          <span style={{ fontSize: 12, fontWeight: 700, color: todayPct >= 100 ? COLORS.accent2 : COLORS.text }}>
            {toFa(todayPct)}٪
          </span>
        </div>
        <div style={{ height: 7, borderRadius: 99, background: COLORS.surface2, overflow: "hidden" }}>
          <div style={{
            height: "100%", width: `${todayPct}%`, borderRadius: 99,
            background: todayPct >= 100 ? COLORS.accent2 : COLORS.accent, transition: "width .4s ease",
          }} />
        </div>
        <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginTop: 8 }}>
          <div style={{ fontSize: 11.5, color: COLORS.muted }}>
            {formatDuration(studiedTodayMin)} از {formatDuration(targetTodayMin)} مطالعه کرده‌ای
          </div>
          <button onClick={() => openGoalEdit("today")} className="kn-btn" style={{
            background: "none", border: `1px solid ${COLORS.border}`, borderRadius: 8,
            padding: "4px 10px", color: COLORS.accent, fontSize: 11, fontWeight: 700,
          }}>تغییر هدف</button>
        </div>

        {goalEditOpen && (
          <div className="kn-fade" style={{
            marginTop: 12, paddingTop: 12, borderTop: `1px solid ${COLORS.border}`,
          }}>
            <div style={{ display: "flex", gap: 6, marginBottom: 10 }}>
              {[
                { id: "today", label: "فقط امروز" },
                { id: "normal", label: "روزهای عادی" },
                { id: "holiday", label: "روزهای تعطیل" },
              ].map((m) => (
                <button key={m.id} onClick={() => openGoalEdit(m.id)} className="kn-btn" style={{
                  flex: 1, border: `1.5px solid ${goalMode === m.id ? COLORS.accent : COLORS.border}`,
                  background: goalMode === m.id ? COLORS.accent + "22" : "transparent",
                  color: goalMode === m.id ? COLORS.accentSoft : COLORS.muted,
                  borderRadius: 8, padding: "6px 0", fontSize: 11, fontWeight: 700,
                }}>{m.label}</button>
              ))}
            </div>
            <div style={{ fontSize: 11.5, color: COLORS.muted, marginBottom: 6 }}>
              {goalMode === "today"
                ? "هدف امروز را مشخص کن"
                : goalMode === "normal"
                  ? "هدف پیش‌فرض روزهای عادی"
                  : "هدف پیش‌فرض جمعه و یکشنبه"}
            </div>
            <div style={{ display: "flex", gap: 8, marginBottom: 8, alignItems: "center" }}>
              <div style={{ flex: 1 }}>
                <div style={{ fontSize: 10, color: COLORS.muted, marginBottom: 4 }}>ساعت</div>
                <input
                  type="number" min="0" max="24" value={goalHoursInput}
                  onChange={(e) => setGoalHoursInput(e.target.value)}
                  placeholder="ساعت"
                  style={{
                    width: "100%", background: COLORS.surface2, border: `1px solid ${COLORS.border}`,
                    borderRadius: 8, padding: "8px 12px", color: COLORS.text, fontSize: 13, outline: "none",
                  }}
                />
              </div>
              <div style={{ flex: 1 }}>
                <div style={{ fontSize: 10, color: COLORS.muted, marginBottom: 4 }}>دقیقه</div>
                <input
                  type="number" min="0" max="59" value={goalMinutesInput}
                  onChange={(e) => setGoalMinutesInput(e.target.value)}
                  placeholder="دقیقه"
                  style={{
                    width: "100%", background: COLORS.surface2, border: `1px solid ${COLORS.border}`,
                    borderRadius: 8, padding: "8px 12px", color: COLORS.text, fontSize: 13, outline: "none",
                  }}
                />
              </div>
              <button onClick={saveGoal} className="kn-btn" style={{
                background: COLORS.accent, color: "#1a1400", border: "none", borderRadius: 8,
                padding: "8px 16px", fontSize: 13, fontWeight: 700, marginTop: 16,
              }}>ذخیره</button>
            </div>
            <div style={{ display: "flex", gap: 8 }}>
              {goalMode === "today" && hasTodayOverride && (
                <button onClick={clearTodayOverride} className="kn-btn" style={{
                  flex: 1, background: "none", border: `1px solid ${COLORS.border}`,
                  borderRadius: 8, padding: "7px 0", color: COLORS.danger, fontSize: 11.5, fontWeight: 600,
                }}>حذف استثنای امروز</button>
              )}
              <button onClick={() => setGoalEditOpen(false)} className="kn-btn" style={{
                flex: 1, background: "none", border: `1px solid ${COLORS.border}`,
                borderRadius: 8, padding: "7px 0", color: COLORS.muted, fontSize: 11.5,
              }}>بستن</button>
            </div>
          </div>
        )}
      </div>

      {/* today's log */}
      {todaySessions.length > 0 && (
        <div style={{ marginBottom: 20 }}>
          <div style={{ fontSize: 13, fontWeight: 700, color: COLORS.muted, marginBottom: 10 }}>جلسات امروز</div>
          <div style={{ display: "flex", flexDirection: "column", gap: 8 }}>
            {todaySessions.map((s) => {
              const subj = curriculumById(s.subjectId);
              const box = boxes.find((b) => b.id === s.boxId);
              return (
                <div key={s.id} style={{
                  background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 12,
                  padding: "10px 14px", display: "flex", alignItems: "center", justifyContent: "space-between",
                }}>
                  <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
                    <span style={{ width: 8, height: 8, borderRadius: 99, background: subj?.color ?? COLORS.muted }} />
                    <div>
                      <div style={{ fontSize: 13, fontWeight: 600 }}>{subj?.name ?? "حذف‌شده"}</div>
                      <div style={{ fontSize: 11, color: COLORS.muted, marginTop: 1 }}>
                        {box ? `باکس ${toFa(box.number)}` : ""}{s.topic ? ` · ${s.topic}` : ""}{s.note ? ` · ${s.note}` : ""}
                      </div>
                      {s.exerciseCount ? (
                        <div style={{ fontSize: 10.5, color: COLORS.accentSoft, marginTop: 1 }}>{toFa(s.exerciseCount)} {s.exerciseType}</div>
                      ) : null}
                    </div>
                  </div>
                  <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
                    <div style={{ fontSize: 12, color: COLORS.muted, fontWeight: 600 }}>{formatDuration(s.minutes)}</div>
                    <button onClick={() => deleteSession(s.id)} style={{ background: "none", border: "none", color: COLORS.muted, padding: 4 }}>
                      <X size={13} />
                    </button>
                  </div>
                </div>
              );
            })}
          </div>
        </div>
      )}

      {/* header + add */}
      <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 14 }}>
        <div style={{ fontSize: 15, fontWeight: 700 }}>باکس‌های مطالعاتی</div>
        <button onClick={() => setAdding((v) => !v)} className="kn-btn" style={{
          background: adding ? COLORS.surface2 : COLORS.accent, color: adding ? COLORS.text : "#1a1400",
          border: "none", borderRadius: 10, padding: "7px 12px", fontSize: 13, fontWeight: 700,
          display: "flex", alignItems: "center", gap: 4,
        }}>
          {adding ? <X size={15} /> : <Plus size={15} />}
          {adding ? "بستن" : "باکس جدید"}
        </button>
      </div>

      {adding && (
        <div className="kn-fade" style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14, padding: 14, marginBottom: 16 }}>
          <div style={{ fontSize: 12, color: COLORS.muted, marginBottom: 10 }}>درس‌هایی که توی این باکس می‌خونی رو انتخاب کن:</div>
          <GroupedSubjectPicker items={CURRICULUM} mode="multi" value={selectedIds} onChange={setSelectedIds} />
          <button onClick={createBox} disabled={selectedIds.length === 0} className="kn-btn" style={{
            marginTop: 6, background: selectedIds.length === 0 ? COLORS.surface2 : COLORS.accent,
            color: selectedIds.length === 0 ? COLORS.muted : "#1a1400", border: "none", borderRadius: 10,
            padding: "9px 16px", fontSize: 13, fontWeight: 700, width: "100%",
          }}>ساخت باکس ({toFa(selectedIds.length)} درس انتخاب‌شده)</button>
        </div>
      )}

      {sortedBoxes.length === 0 && !adding && (
        <div style={{ fontSize: 13, color: COLORS.muted, textAlign: "center", padding: 40 }}>امروز هنوز باکسی نساختی. (هر روز باکس‌ها از نو شروع می‌شوند)</div>
      )}

      <div style={{ display: "flex", flexDirection: "column", gap: 10 }}>
        {sortedBoxes.map((box) => (
          <BoxCard key={box.id} box={box} sessions={sessions} setSessions={setSessions} onDelete={() => deleteBox(box.id)} />
        ))}
      </div>
    </div>
  );
}


// ---------- چرخونه انتخاب زمان (شبیه آلارم) ----------
function WheelColumn({ items, value, onChange, label }) {
  const itemH = 36;
  const visible = 5;
  const ref = useRef(null);
  const scrolling = useRef(false);

  useEffect(() => {
    const el = ref.current;
    if (!el) return;
    const idx = Math.max(0, items.indexOf(value));
    el.scrollTop = idx * itemH;
  }, []); // فقط mount

  useEffect(() => {
    const el = ref.current;
    if (!el || scrolling.current) return;
    const idx = Math.max(0, items.indexOf(value));
    const target = idx * itemH;
    if (Math.abs(el.scrollTop - target) > 2) el.scrollTop = target;
  }, [value, items]);

  function onScroll() {
    const el = ref.current;
    if (!el) return;
    scrolling.current = true;
    const idx = Math.round(el.scrollTop / itemH);
    const clamped = Math.max(0, Math.min(items.length - 1, idx));
    if (items[clamped] !== value) onChange(items[clamped]);
  }

  function onScrollEnd() {
    const el = ref.current;
    if (!el) return;
    const idx = Math.round(el.scrollTop / itemH);
    const clamped = Math.max(0, Math.min(items.length - 1, idx));
    el.scrollTo({ top: clamped * itemH, behavior: "smooth" });
    onChange(items[clamped]);
    setTimeout(() => { scrolling.current = false; }, 120);
  }

  return (
    <div style={{ flex: 1, textAlign: "center" }}>
      <div style={{ fontSize: 11, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>{label}</div>
      <div style={{ position: "relative", height: itemH * visible, overflow: "hidden" }}>
        <div style={{
          pointerEvents: "none", position: "absolute", left: 4, right: 4, top: itemH * 2,
          height: itemH, borderRadius: 10, border: `1.5px solid ${COLORS.accent}66`,
          background: COLORS.accent + "14", zIndex: 1,
        }} />
        <div style={{
          pointerEvents: "none", position: "absolute", left: 0, right: 0, top: 0, height: itemH * 2,
          background: `linear-gradient(to bottom, ${COLORS.surface2}, transparent)`, zIndex: 2,
        }} />
        <div style={{
          pointerEvents: "none", position: "absolute", left: 0, right: 0, bottom: 0, height: itemH * 2,
          background: `linear-gradient(to top, ${COLORS.surface2}, transparent)`, zIndex: 2,
        }} />
        <div
          ref={ref}
          onScroll={onScroll}
          onTouchEnd={onScrollEnd}
          onMouseUp={onScrollEnd}
          style={{
            height: itemH * visible, overflowY: "auto", scrollSnapType: "y mandatory",
            WebkitOverflowScrolling: "touch", paddingTop: itemH * 2, paddingBottom: itemH * 2,
          }}
        >
          {items.map((it) => (
            <div key={it} style={{
              height: itemH, display: "flex", alignItems: "center", justifyContent: "center",
              scrollSnapAlign: "center", fontSize: value === it ? 18 : 14,
              fontWeight: value === it ? 800 : 500,
              color: value === it ? COLORS.accentSoft : COLORS.muted,
            }}>{toFa(it)}</div>
          ))}
        </div>
      </div>
    </div>
  );
}

function DurationWheel({ hours, minutes, onHours, onMinutes }) {
  const hourItems = Array.from({ length: 13 }, (_, i) => i); // 0..12
  const minuteItems = [0, 5, 10, 15, 20, 25, 30, 35, 40, 45, 50, 55];
  const h = hourItems.includes(Number(hours)) ? Number(hours) : 0;
  let m = Number(minutes) || 0;
  // نزدیک‌ترین ۵ دقیقه
  m = minuteItems.reduce((best, x) => Math.abs(x - m) < Math.abs(best - m) ? x : best, 0);

  return (
    <div style={{
      display: "flex", gap: 8, background: COLORS.bg, border: `1px solid ${COLORS.border}`,
      borderRadius: 12, padding: "10px 8px", marginBottom: 10,
    }}>
      <WheelColumn items={hourItems} value={h} onChange={(v) => onHours(String(v))} label="ساعت" />
      <div style={{ width: 12, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 20, fontWeight: 800, color: COLORS.muted, paddingTop: 18 }}>:</div>
      <WheelColumn items={minuteItems} value={m} onChange={(v) => onMinutes(String(v))} label="دقیقه" />
    </div>
  );
}

/** چرخونه تک‌ستونه بر حسب دقیقه (گام ۵، حداکثر ۵ ساعت = ۳۰۰ دقیقه) */
function MinutesWheel({ minutes, onChange, label = "دقیقه", maxMin = 300 }) {
  const items = [];
  for (let i = 0; i <= maxMin; i += 5) items.push(i);
  let m = Number(minutes) || 0;
  m = items.reduce((best, x) => Math.abs(x - m) < Math.abs(best - m) ? x : best, 0);
  return (
    <div style={{
      background: COLORS.bg, border: `1px solid ${COLORS.border}`,
      borderRadius: 12, padding: "10px 8px", marginBottom: 10,
    }}>
      <WheelColumn items={items} value={m} onChange={(v) => onChange(String(v))} label={label} />
      <div style={{ textAlign: "center", fontSize: 11, color: COLORS.muted, marginTop: 4 }}>
        {m >= 60 ? formatDuration(m) : `${toFa(m)} دقیقه`}
      </div>
    </div>
  );
}

function BoxCard({ box, sessions, setSessions, onDelete }) {
  const [logOpen, setLogOpen] = useState(false);
  const boxSubjects = box.subjectIds.map((id) => curriculumById(id)).filter(Boolean);
  const [subjectId, setSubjectId] = useState(boxSubjects[0]?.id ?? "");
  const [hours, setHours] = useState("1");
  const [minutes, setMinutes] = useState("15");
  const [date, setDate] = useState(todayKey());
  const [topic, setTopic] = useState("");
  const [selectedTopics, setSelectedTopics] = useState([]); // multi from chapter picker
  const [topicSheetOpen, setTopicSheetOpen] = useState(false);
  const lastTapRef = useRef({ id: null, t: 0 });
  const [exerciseType, setExerciseType] = useState("تمرین");
  const [exerciseCount, setExerciseCount] = useState("");
  const [note, setNote] = useState("");

  const boxSessions = sessions.filter((s) => s.boxId === box.id).sort((a, b) => b.timestamp - a.timestamp);
  const totalMin = boxSessions.reduce((a, s) => a + s.minutes, 0);
  const recentTopics = useMemo(() => {
    const seen = [];
    sessions
      .filter((s) => s.subjectId === subjectId && s.topic)
      .sort((a, b) => b.timestamp - a.timestamp)
      .forEach((s) => { if (!seen.includes(s.topic)) seen.push(s.topic); });
    return seen.slice(0, 6);
  }, [sessions, subjectId]);

  function onSubjectTap(id) {
    const now = Date.now();
    const last = lastTapRef.current;
    if (last.id === id && now - last.t < 350) {
      setSubjectId(id);
      setTopicSheetOpen(true);
      lastTapRef.current = { id: null, t: 0 };
      return;
    }
    lastTapRef.current = { id, t: now };
    setSubjectId(id);
    if (id !== subjectId) {
      setSelectedTopics([]);
      setTopic("");
    }
  }

  function toggleChapterTopic(label) {
    setSelectedTopics((prev) => {
      if (prev.includes(label)) return prev.filter((x) => x !== label);
      return [...prev, label];
    });
  }

  function applyTopicsFromSheet() {
    setTopic(selectedTopics.join(" · "));
    setTopicSheetOpen(false);
  }

  function submit() {
    const h = parseInt(hours || "0", 10);
    const m = parseInt(minutes || "0", 10);
    const total = h * 60 + m;
    if (!subjectId || total < 1) return;
    const ec = exerciseCount.trim() !== "" ? parseInt(exerciseCount, 10) : null;
    const topicStr = selectedTopics.length ? selectedTopics.join(" · ") : topic.trim();
    const s = {
      id: Date.now().toString(), boxId: box.id, subjectId, minutes: total,
      date: date || todayKey(), topic: topicStr, topics: selectedTopics.length ? [...selectedTopics] : (topicStr ? [topicStr] : []),
      note: note.trim(),
      exerciseType: ec ? exerciseType : null, exerciseCount: ec, timestamp: Date.now(),
    };
    setSessions((prev) => [...prev, s]);
    setHours("1"); setMinutes("15"); setNote(""); setTopic(""); setSelectedTopics([]); setExerciseCount("");
    setLogOpen(false);
  }

  return (
    <div style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14, overflow: "hidden" }}>
      <div style={{ padding: "13px 14px" }}>
        <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 10 }}>
          <div>
            <div style={{ fontSize: 14, fontWeight: 700 }}>باکس {toFa(box.number)}</div>
            <div style={{ fontSize: 11, color: COLORS.muted, marginTop: 1 }}>
              {toJalaliShort(new Date(box.dayKey + "T00:00:00"))} · {formatDuration(totalMin)} مطالعه‌شده
            </div>
          </div>
          <button onClick={onDelete} className="kn-btn" style={{ background: "none", border: "none", color: COLORS.muted, padding: 6 }}>
            <Trash2 size={16} />
          </button>
        </div>
        <div style={{ display: "flex", gap: 6, flexWrap: "wrap", marginBottom: 12 }}>
          {boxSubjects.map((s) => (
            <span key={s.id} style={{
              fontSize: 11, padding: "3px 9px", borderRadius: 999, background: s.color + "22", color: s.color, fontWeight: 600,
            }}>{s.name}</span>
          ))}
        </div>
        <button onClick={() => setLogOpen((v) => !v)} className="kn-btn" style={{
          background: logOpen ? COLORS.surface2 : COLORS.accent, color: logOpen ? COLORS.text : "#1a1400",
          border: "none", borderRadius: 10, padding: "8px 12px", fontSize: 12.5, fontWeight: 700,
          display: "flex", alignItems: "center", gap: 4, width: "100%", justifyContent: "center",
        }}>
          {logOpen ? <X size={14} /> : <Plus size={14} />}
          {logOpen ? "بستن" : "ثبت زمان مطالعه"}
        </button>
      </div>

      {logOpen && (
        <div className="kn-fade" style={{ borderTop: `1px solid ${COLORS.border}`, padding: 12, background: COLORS.surface2 }}>
          <div style={{ fontSize: 10.5, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>
            درس <span style={{ fontWeight: 500 }}>(دوبار بزن برای انتخاب مبحث)</span>
          </div>
          <div style={{ display: "flex", gap: 6, flexWrap: "wrap", marginBottom: 10 }}>
            {boxSubjects.map((s) => (
              <button key={s.id} onClick={() => onSubjectTap(s.id)} className="kn-btn" style={{
                border: `1.5px solid ${subjectId === s.id ? s.color : COLORS.border}`,
                background: subjectId === s.id ? s.color + "22" : "transparent",
                color: subjectId === s.id ? COLORS.text : COLORS.muted,
                borderRadius: 999, padding: "5px 11px", fontSize: 12, fontWeight: 600,
              }}>{s.name}</button>
            ))}
          </div>

          {(selectedTopics.length > 0 || topic) && (
            <div style={{ display: "flex", gap: 5, flexWrap: "wrap", marginBottom: 8 }}>
              {(selectedTopics.length ? selectedTopics : [topic]).filter(Boolean).map((t) => (
                <span key={t} style={{
                  fontSize: 11, padding: "3px 9px", borderRadius: 999,
                  background: COLORS.accent + "22", color: COLORS.accentSoft, fontWeight: 600,
                }}>{t}</span>
              ))}
            </div>
          )}

          <input
            value={topic} onChange={(e) => { setTopic(e.target.value); setSelectedTopics([]); }}
            placeholder="مبحث (دستی یا دوبار روی درس)"
            style={{ width: "100%", background: COLORS.bg, border: `1px solid ${COLORS.border}`, borderRadius: 8, padding: "7px 10px", color: COLORS.text, fontSize: 12.5, outline: "none", marginBottom: 8 }}
          />
          {recentTopics.length > 0 && (
            <div style={{ display: "flex", gap: 5, flexWrap: "wrap", marginBottom: 10 }}>
              {recentTopics.map((t) => (
                <button key={t} onClick={() => { setTopic(t); setSelectedTopics([]); }} className="kn-btn" style={{
                  background: topic === t ? COLORS.accent2 + "22" : COLORS.bg,
                  color: topic === t ? COLORS.accent2 : COLORS.muted,
                  border: `1px solid ${topic === t ? COLORS.accent2 : COLORS.border}`,
                  borderRadius: 999, padding: "3px 9px", fontSize: 10.5,
                }}>{t}</button>
              ))}
            </div>
          )}

          {topicSheetOpen && (
            <div style={{
              position: "fixed", inset: 0, background: "rgba(0,0,0,0.55)", zIndex: 90,
              display: "flex", alignItems: "flex-end", justifyContent: "center",
            }} onClick={() => setTopicSheetOpen(false)}>
              <div onClick={(e) => e.stopPropagation()} style={{
                width: "100%", maxWidth: 480, maxHeight: "75vh", overflowY: "auto",
                background: COLORS.surface, borderRadius: "18px 18px 0 0", padding: 16,
                border: `1px solid ${COLORS.border}`,
              }}>
                <div style={{ fontSize: 15, fontWeight: 800, marginBottom: 4 }}>
                  مباحث {curriculumById(subjectId)?.name || ""}
                </div>
                <div style={{ fontSize: 12, color: COLORS.muted, marginBottom: 12 }}>
                  چند مبحث را می‌توانی همزمان انتخاب کنی
                </div>
                <div style={{ display: "flex", flexDirection: "column", gap: 6, marginBottom: 14 }}>
                  {chaptersOf(subjectId).map((ch) => {
                    const label = isLessonBasedSubject(subjectId)
                      ? ch.title
                      : `فصل ${toFa(ch.num)} — ${ch.title}`;
                    const on = selectedTopics.includes(label);
                    return (
                      <button key={ch.num} type="button" onClick={() => toggleChapterTopic(label)} className="kn-btn" style={{
                        textAlign: "right",
                        border: `1.5px solid ${on ? COLORS.accent : COLORS.border}`,
                        background: on ? COLORS.accent + "18" : COLORS.bg,
                        color: on ? COLORS.accentSoft : COLORS.text,
                        borderRadius: 10, padding: "10px 12px", fontSize: 12.5, fontWeight: 600,
                      }}>
                        {on ? "✓ " : ""}{label}
                      </button>
                    );
                  })}
                </div>
                <div style={{ display: "flex", gap: 8 }}>
                  <button onClick={applyTopicsFromSheet} className="kn-btn" style={{
                    flex: 1, background: COLORS.accent, color: "#1a1400", border: "none", borderRadius: 10, padding: "11px 0", fontWeight: 800,
                  }}>تأیید ({toFa(selectedTopics.length)} مبحث)</button>
                  <button onClick={() => setTopicSheetOpen(false)} className="kn-btn" style={{
                    flex: 1, background: "none", border: `1px solid ${COLORS.border}`, borderRadius: 10, padding: "11px 0", color: COLORS.muted,
                  }}>بستن</button>
                </div>
              </div>
            </div>
          )}

          <div style={{ fontSize: 10.5, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>مدت مطالعه</div>
          <DurationWheel hours={hours || "0"} minutes={minutes || "0"} onHours={setHours} onMinutes={setMinutes} />

          <div style={{ fontSize: 10.5, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>تاریخ</div>
          <div style={{ marginBottom: 10 }}>
            <JalaliDatePicker value={date} onChange={setDate} />
          </div>

          <div style={{ fontSize: 10.5, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>چند تا تست یا تمرین زدی؟ (اختیاری)</div>
          <div style={{ display: "flex", gap: 6, marginBottom: 8 }}>
            <button type="button" onClick={() => setExerciseType("تمرین")} className="kn-btn" style={{
              flex: 1, background: exerciseType === "تمرین" ? COLORS.accent2 + "22" : COLORS.bg,
              color: exerciseType === "تمرین" ? COLORS.accent2 : COLORS.muted,
              border: `1px solid ${exerciseType === "تمرین" ? COLORS.accent2 : COLORS.border}`,
              borderRadius: 8, padding: "6px 0", fontSize: 12, fontWeight: 600,
            }}>تمرین</button>
            <button type="button" onClick={() => setExerciseType("تست")} className="kn-btn" style={{
              flex: 1, background: exerciseType === "تست" ? COLORS.accent + "22" : COLORS.bg,
              color: exerciseType === "تست" ? COLORS.accent : COLORS.muted,
              border: `1px solid ${exerciseType === "تست" ? COLORS.accent : COLORS.border}`,
              borderRadius: 8, padding: "6px 0", fontSize: 12, fontWeight: 600,
            }}>تست</button>
          </div>
          <input
            type="number" min="0" value={exerciseCount} onChange={(e) => setExerciseCount(e.target.value)}
            placeholder={`تعداد ${exerciseType}`}
            style={{ width: "100%", background: COLORS.bg, border: `1px solid ${COLORS.border}`, borderRadius: 8, padding: "7px 10px", color: COLORS.text, fontSize: 12.5, outline: "none", marginBottom: 10 }}
          />

          <input value={note} onChange={(e) => setNote(e.target.value)} placeholder="یادداشت (اختیاری)"
            style={{ width: "100%", background: COLORS.bg, border: `1px solid ${COLORS.border}`, borderRadius: 8, padding: "7px 10px", color: COLORS.text, fontSize: 12.5, outline: "none", marginBottom: 10 }} />
          <button onClick={submit} className="kn-btn" style={{
            background: COLORS.accent2, color: "#062420", border: "none", borderRadius: 8, padding: "8px 12px", fontSize: 12.5, fontWeight: 700, width: "100%",
          }}>ثبت و افزودن به لیست</button>

          {boxSessions.length > 0 && (
            <div style={{ marginTop: 12 }}>
              <div style={{ fontSize: 10.5, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>جلسات این باکس</div>
              <div style={{ display: "flex", flexDirection: "column", gap: 5 }}>
                {boxSessions.slice(0, 5).map((s) => {
                  const subj = curriculumById(s.subjectId);
                  return (
                    <div key={s.id} style={{ fontSize: 11.5, color: COLORS.muted }}>
                      <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between" }}>
                        <span>{subj?.name ?? "—"}{s.topic ? ` · ${s.topic}` : ""}</span>
                        <span>{formatDuration(s.minutes)}</span>
                      </div>
                      {s.exerciseCount ? (
                        <div style={{ fontSize: 10.5, color: COLORS.accentSoft, marginTop: 1 }}>
                          {toFa(s.exerciseCount)} {s.exerciseType}
                        </div>
                      ) : null}
                    </div>
                  );
                })}
              </div>
            </div>
          )}
        </div>
      )}
    </div>
  );
}

// ================= HOMEWORK =================


function HomeworkView({ profile }) {
  const [sharedTasks, setSharedTasks] = useState([]);
  const [progressMap, setProgressMap] = useState({});
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState("");
  const [text, setText] = useState("");
  const [subjectId, setSubjectId] = useState(CURRICULUM[0].id);
  const [topic, setTopic] = useState("");
  const [type, setType] = useState("تمرین");
  const [targetCount, setTargetCount] = useState("");
  const [qtyMode, setQtyMode] = useState("count"); // count | range
  const [rangeFrom, setRangeFrom] = useState("");
  const [rangeTo, setRangeTo] = useState("");
  const [rangeStep, setRangeStep] = useState("1");
  const [dueDate, setDueDate] = useState(todayKey());
  const [showDone, setShowDone] = useState(false);
  const [busy, setBusy] = useState(false);

  const todayStr = todayKey();
  const userId = profile?.userId;

  async function reload() {
    setLoading(true);
    setError("");
    try {
      const [tasks, prog] = await Promise.all([
        fetchSharedTasks(),
        fetchUserTaskProgress(userId),
      ]);
      setSharedTasks(tasks);
      setProgressMap(prog || {});
    } catch (e) {
      console.error(e);
      setError("خطا در دریافت تکالیف. اتصال اینترنت و قوانین Firestore را چک کن.");
    }
    setLoading(false);
  }

  useEffect(() => { reload(); }, [userId]);

  const recentTopics = useMemo(() => {
    const seen = [];
    sharedTasks
      .filter((t) => t.subjectId === subjectId && t.topic)
      .forEach((t) => { if (!seen.includes(t.topic)) seen.push(t.topic); });
    return seen.slice(0, 6);
  }, [sharedTasks, subjectId]);

  function mergeTask(t) {
    const p = progressMap[t.id] || {};
    const doneCount = p.doneCount || 0;
    const done = p.done === true || (t.targetCount ? doneCount >= t.targetCount : false);
    return {
      ...t,
      done,
      doneCount,
      completedAt: p.completedAt || null,
    };
  }

  function remainingOf(t) {
    if (t.targetCount && t.targetCount > 0) {
      return Math.max(0, t.targetCount - (t.doneCount || 0));
    }
    return t.done ? 0 : 1;
  }

  function dueKeyOf(t) {
    return t.dueDate || t.date || "9999-12-31";
  }

  const merged = useMemo(() => sharedTasks.map(mergeTask), [sharedTasks, progressMap]);

  const remainingTasks = useMemo(() => {
    return merged
      .filter((t) => !t.done)
      .slice()
      .sort((a, b) => {
        const da = dueKeyOf(a);
        const db = dueKeyOf(b);
        if (da !== db) return da < db ? -1 : 1;
        const ra = remainingOf(a);
        const rb = remainingOf(b);
        if (rb !== ra) return rb - ra;
        return (b.createdAt || 0) - (a.createdAt || 0);
      });
  }, [merged]);

  const doneTasks = useMemo(() => {
    return merged
      .filter((t) => t.done)
      .slice()
      .sort((a, b) => (b.completedAt || 0) - (a.completedAt || 0));
  }, [merged]);

  function calcRangeCount(from, to, step) {
    const a = parseInt(from, 10);
    const b = parseInt(to, 10);
    const s = parseInt(step, 10) || 1;
    if (isNaN(a) || isNaN(b) || s < 1) return null;
    if (b < a) return null;
    return Math.floor((b - a) / s) + 1;
  }

  const computedRangeCount = qtyMode === "range"
    ? calcRangeCount(rangeFrom, rangeTo, rangeStep)
    : null;

  async function addTask() {
    if (!text.trim() || !subjectId || !userId) return;
    setBusy(true);
    try {
      let tc = null;
      let rf = null, rt = null, rs = null;
      if (qtyMode === "range") {
        tc = calcRangeCount(rangeFrom, rangeTo, rangeStep);
        if (tc == null || tc < 1) {
          setError("بازه تست نامعتبر است (اول ≤ آخر و ضریب ≥ ۱)");
          setBusy(false);
          return;
        }
        rf = parseInt(rangeFrom, 10);
        rt = parseInt(rangeTo, 10);
        rs = parseInt(rangeStep, 10) || 1;
      } else if (targetCount.trim() !== "") {
        tc = parseInt(targetCount, 10);
        if (isNaN(tc) || tc < 1) tc = null;
      }
      const task = {
        id: Date.now().toString(36) + Math.random().toString(36).slice(2, 7),
        subjectId,
        text: text.trim(),
        topic: topic.trim(),
        type,
        targetCount: tc,
        rangeFrom: rf,
        rangeTo: rt,
        rangeStep: rs,
        dueDate: dueDate || todayStr,
        date: todayStr,
        createdAt: Date.now(),
        createdBy: userId,
        createdByName: profile?.name || profile?.username || "",
      };
      await createSharedTask(task);
      setSharedTasks((prev) => [task, ...prev]);
      setText("");
      setTopic("");
      setTargetCount("");
      setRangeFrom("");
      setRangeTo("");
      setRangeStep("1");
      setDueDate(todayKey());
      setError("");
    } catch (e) {
      console.error(e);
      setError("ثبت تکلیف ناموفق بود");
    }
    setBusy(false);
  }

  async function toggleTask(t) {
    if (!userId) return;
    const nextDone = !t.done;
    const progress = {
      done: nextDone,
      doneCount: nextDone && t.targetCount ? t.targetCount : (t.doneCount || 0),
      completedAt: nextDone ? Date.now() : null,
    };
    setProgressMap((prev) => ({ ...prev, [t.id]: { ...(prev[t.id] || {}), ...progress } }));
    try {
      await saveUserTaskProgress(userId, t.id, progress);
    } catch (e) {
      console.error(e);
      setError("ذخیره پیشرفت ناموفق بود");
    }
  }

  async function deleteTask(t) {
    // فقط سازنده یا ادمین می‌تواند تکلیف مشترک را حذف کند
    const canDelete = profile?.role === "admin" || t.createdBy === userId;
    if (!canDelete) {
      setError("فقط سازنده تکلیف یا ادمین می‌تواند آن را حذف کند");
      return;
    }
    if (!confirm("این تکلیف برای همه حذف شود؟")) return;
    try {
      await deleteSharedTask(t.id);
      setSharedTasks((prev) => prev.filter((x) => x.id !== t.id));
    } catch (e) {
      console.error(e);
      setError("حذف ناموفق بود");
    }
  }

  async function addProgress(t, count) {
    if (!userId || !count || count <= 0) return;
    const doneCount = (t.doneCount || 0) + count;
    const done = t.targetCount ? doneCount >= t.targetCount : t.done;
    const progress = {
      done,
      doneCount,
      completedAt: done ? Date.now() : null,
    };
    setProgressMap((prev) => ({ ...prev, [t.id]: { ...(prev[t.id] || {}), ...progress } }));
    try {
      await saveUserTaskProgress(userId, t.id, progress);
    } catch (e) {
      console.error(e);
      setError("ذخیره پیشرفت ناموفق بود");
    }
  }

  return (
    <div className="kn-fade">
      <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 4 }}>
        <div style={{ fontSize: 15, fontWeight: 700 }}>تکالیف مشترک</div>
        <button onClick={reload} className="kn-btn" style={{
          background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 8,
          padding: "5px 10px", color: COLORS.muted, fontSize: 11.5, fontWeight: 600,
        }}>بروزرسانی</button>
      </div>
      <div style={{ fontSize: 12, color: COLORS.muted, marginBottom: 14, lineHeight: 1.7 }}>
        تکالیف برای همه یکسان است؛ پیشرفت و انجام‌شدن فقط مال خودت است.
        {remainingTasks.length > 0 ? ` · ${toFa(remainingTasks.length)} مورد باز` : ""}
      </div>

      {error && (
        <div style={{ fontSize: 12, color: COLORS.danger, background: COLORS.danger + "18", borderRadius: 10, padding: "8px 12px", marginBottom: 12 }}>
          {error}
        </div>
      )}

      {/* add form */}
      <div style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14, padding: 14, marginBottom: 16 }}>
        <div style={{ marginBottom: 10 }}>
          <GroupedSubjectPicker items={CURRICULUM} mode="single" value={subjectId} onChange={setSubjectId} />
        </div>

        <input
          value={topic} onChange={(e) => setTopic(e.target.value)} placeholder="مبحث (اختیاری، مثلاً فصل ۲ - مشتق)"
          style={{ width: "100%", background: COLORS.surface2, border: `1px solid ${COLORS.border}`, borderRadius: 10, padding: "8px 12px", color: COLORS.text, fontSize: 12.5, outline: "none", marginBottom: 8 }}
        />
        {recentTopics.length > 0 && (
          <div style={{ display: "flex", gap: 5, flexWrap: "wrap", marginBottom: 10 }}>
            {recentTopics.map((t) => (
              <button key={t} onClick={() => setTopic(t)} className="kn-btn" style={{
                background: topic === t ? COLORS.accent2 + "22" : COLORS.surface2,
                color: topic === t ? COLORS.accent2 : COLORS.muted,
                border: `1px solid ${topic === t ? COLORS.accent2 : COLORS.border}`,
                borderRadius: 999, padding: "3px 9px", fontSize: 10.5,
              }}>{t}</button>
            ))}
          </div>
        )}

        <div style={{ display: "flex", gap: 6, marginBottom: 8 }}>
          <button type="button" onClick={() => setType("تمرین")} className="kn-btn" style={{
            flex: 1, background: type === "تمرین" ? COLORS.accent2 + "22" : COLORS.surface2,
            color: type === "تمرین" ? COLORS.accent2 : COLORS.muted,
            border: `1px solid ${type === "تمرین" ? COLORS.accent2 : COLORS.border}`,
            borderRadius: 8, padding: "6px 0", fontSize: 12, fontWeight: 600,
          }}>تمرین</button>
          <button type="button" onClick={() => setType("تست")} className="kn-btn" style={{
            flex: 1, background: type === "تست" ? COLORS.accent + "22" : COLORS.surface2,
            color: type === "تست" ? COLORS.accent : COLORS.muted,
            border: `1px solid ${type === "تست" ? COLORS.accent : COLORS.border}`,
            borderRadius: 8, padding: "6px 0", fontSize: 12, fontWeight: 600,
          }}>تست</button>
        </div>

        <div style={{ display: "flex", gap: 6, marginBottom: 8 }}>
          <button type="button" onClick={() => setQtyMode("count")} className="kn-btn" style={{
            flex: 1, border: `1.5px solid ${qtyMode === "count" ? COLORS.accent : COLORS.border}`,
            background: qtyMode === "count" ? COLORS.accent + "22" : "transparent",
            color: qtyMode === "count" ? COLORS.accentSoft : COLORS.muted,
            borderRadius: 8, padding: "6px 0", fontSize: 11.5, fontWeight: 700,
          }}>تعداد مستقیم</button>
          <button type="button" onClick={() => setQtyMode("range")} className="kn-btn" style={{
            flex: 1, border: `1.5px solid ${qtyMode === "range" ? COLORS.accent : COLORS.border}`,
            background: qtyMode === "range" ? COLORS.accent + "22" : "transparent",
            color: qtyMode === "range" ? COLORS.accentSoft : COLORS.muted,
            borderRadius: 8, padding: "6px 0", fontSize: 11.5, fontWeight: 700,
          }}>از … تا … با ضریب</button>
        </div>

        {qtyMode === "count" ? (
          <input
            type="number" min="0" value={targetCount} onChange={(e) => setTargetCount(e.target.value)}
            placeholder={`تعداد ${type} (اختیاری)`}
            style={{ width: "100%", background: COLORS.surface2, border: `1px solid ${COLORS.border}`, borderRadius: 8, padding: "8px 10px", color: COLORS.text, fontSize: 12, outline: "none", marginBottom: 8 }}
          />
        ) : (
          <div style={{ marginBottom: 8 }}>
            <div style={{ display: "flex", gap: 6, marginBottom: 6 }}>
              <div style={{ flex: 1 }}>
                <div style={{ fontSize: 10, color: COLORS.muted, marginBottom: 3 }}>تست اول</div>
                <input type="number" value={rangeFrom} onChange={(e) => setRangeFrom(e.target.value)} placeholder="مثلاً ۱۰۰۰"
                  style={{ width: "100%", background: COLORS.surface2, border: `1px solid ${COLORS.border}`, borderRadius: 8, padding: "7px 8px", color: COLORS.text, fontSize: 12, outline: "none" }} />
              </div>
              <div style={{ flex: 1 }}>
                <div style={{ fontSize: 10, color: COLORS.muted, marginBottom: 3 }}>تست آخر</div>
                <input type="number" value={rangeTo} onChange={(e) => setRangeTo(e.target.value)} placeholder="مثلاً ۲۰۰۰"
                  style={{ width: "100%", background: COLORS.surface2, border: `1px solid ${COLORS.border}`, borderRadius: 8, padding: "7px 8px", color: COLORS.text, fontSize: 12, outline: "none" }} />
              </div>
              <div style={{ flex: 0.85 }}>
                <div style={{ fontSize: 10, color: COLORS.muted, marginBottom: 3 }}>ضریب</div>
                <input type="number" min="1" value={rangeStep} onChange={(e) => setRangeStep(e.target.value)} placeholder="۴"
                  style={{ width: "100%", background: COLORS.surface2, border: `1px solid ${COLORS.border}`, borderRadius: 8, padding: "7px 8px", color: COLORS.text, fontSize: 12, outline: "none" }} />
              </div>
            </div>
            <div style={{ fontSize: 11.5, color: COLORS.accentSoft, fontWeight: 600 }}>
              {computedRangeCount != null
                ? `تعداد محاسبه‌شده: ${toFa(computedRangeCount)} ${type}`
                : "تست اول، آخر و ضریب را وارد کن"}
            </div>
          </div>
        )}

        <div style={{ fontSize: 11, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>تاریخ تحویل</div>
        <div style={{ marginBottom: 10 }}>
          <JalaliDatePicker value={dueDate} onChange={setDueDate} />
        </div>

        <div style={{ display: "flex", gap: 8 }}>
          <input
            value={text} onChange={(e) => setText(e.target.value)}
            onKeyDown={(e) => e.key === "Enter" && addTask()}
            placeholder="عنوان تکلیف (برای همه)..."
            style={{ flex: 1, background: COLORS.surface2, border: `1px solid ${COLORS.border}`, borderRadius: 10, padding: "9px 12px", color: COLORS.text, fontSize: 13, outline: "none" }}
          />
          <button onClick={addTask} disabled={busy} className="kn-btn" style={{
            background: COLORS.accent, border: "none", borderRadius: 10, padding: "0 16px", color: "#1a1400",
          }}><Plus size={17} /></button>
        </div>
      </div>

      {loading ? (
        <div style={{ color: COLORS.muted, textAlign: "center", padding: 30 }}>در حال بارگذاری تکالیف...</div>
      ) : remainingTasks.length === 0 ? (
        <div style={{ fontSize: 13, color: COLORS.muted, background: COLORS.surface, borderRadius: 12, padding: 20, textAlign: "center", marginBottom: 16 }}>
          تکلیف باقی‌مانده‌ای نداری 🎉
        </div>
      ) : (
        <div style={{ display: "flex", flexDirection: "column", gap: 8, marginBottom: 16 }}>
          {remainingTasks.map((t) => (
            <TaskRow
              key={t.id} t={t} todayStr={todayStr}
              canDelete={profile?.role === "admin" || t.createdBy === userId}
              onToggle={() => toggleTask(t)}
              onDelete={() => deleteTask(t)}
              onAddProgress={(count) => addProgress(t, count)}
            />
          ))}
        </div>
      )}

      {doneTasks.length > 0 && (
        <div>
          <button onClick={() => setShowDone((v) => !v)} className="kn-btn" style={{
            background: "none", border: "none", color: COLORS.muted, fontSize: 12.5, fontWeight: 700, padding: "4px 0", marginBottom: 8,
          }}>
            {showDone ? "▼" : "◀"} انجام‌شده‌های من ({toFa(doneTasks.length)})
          </button>
          {showDone && (
            <div style={{ display: "flex", flexDirection: "column", gap: 8 }}>
              {doneTasks.map((t) => (
                <TaskRow
                  key={t.id} t={t} todayStr={todayStr}
                  canDelete={profile?.role === "admin" || t.createdBy === userId}
                  onToggle={() => toggleTask(t)}
                  onDelete={() => deleteTask(t)}
                  onAddProgress={(count) => addProgress(t, count)}
                />
              ))}
            </div>
          )}
        </div>
      )}
    </div>
  );
}

function TaskRow({ t, onToggle, onDelete, onAddProgress, todayStr, canDelete = true }) {
  const [progOpen, setProgOpen] = useState(false);
  const [progVal, setProgVal] = useState("");
  const subj = curriculumById(t.subjectId);
  const due = t.dueDate || t.date;
  const remaining = t.targetCount
    ? Math.max(0, t.targetCount - (t.doneCount || 0))
    : (t.done ? 0 : 1);
  const isOverdue = !t.done && due && due < (todayStr || todayKey());
  const isToday = due === (todayStr || todayKey());
  let dueLabel = "";
  if (due) {
    try {
      dueLabel = toJalaliShort(new Date(due + "T00:00:00"));
    } catch (e) {
      dueLabel = due;
    }
  }

  return (
    <div style={{
      background: COLORS.surface,
      border: `1px solid ${isOverdue ? COLORS.danger + "66" : COLORS.border}`,
      borderRadius: 12, padding: "10px 14px",
      opacity: t.done ? 0.7 : 1,
    }}>
      <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", gap: 8 }}>
        <button onClick={onToggle} className="kn-btn" style={{
          background: "none", border: "none", display: "flex", alignItems: "center", gap: 8, flex: 1, textAlign: "right",
          color: t.done ? COLORS.muted : COLORS.text, fontSize: 13, textDecoration: t.done ? "line-through" : "none",
        }}>
          {t.done ? <CheckCircle2 size={16} color={COLORS.accent2} /> : <Circle size={16} />}
          <span style={{ width: 7, height: 7, borderRadius: 99, background: subj?.color ?? COLORS.muted, flexShrink: 0 }} />
          {t.text}
        </button>
        {canDelete && (
          <button onClick={onDelete} style={{ background: "none", border: "none", color: COLORS.muted, padding: 4 }}>
            <X size={14} />
          </button>
        )}
      </div>
      <div style={{ display: "flex", alignItems: "center", flexWrap: "wrap", gap: 6, marginTop: 6, marginRight: 24 }}>
        {due && (
          <span style={{
            fontSize: 10.5, fontWeight: 700, padding: "2px 8px", borderRadius: 999,
            background: isOverdue ? COLORS.danger + "22" : isToday ? COLORS.accent + "22" : COLORS.surface2,
            color: isOverdue ? COLORS.danger : isToday ? COLORS.accentSoft : COLORS.muted,
          }}>
            تحویل: {dueLabel}{isOverdue ? " · گذشته" : isToday ? " · امروز" : ""}
          </span>
        )}
        {t.topic && <span style={{ fontSize: 10.5, color: COLORS.muted }}>📌 {t.topic}</span>}
        {t.type && (
          <span style={{
            fontSize: 10, fontWeight: 700, padding: "2px 7px", borderRadius: 999,
            background: t.type === "تست" ? COLORS.accent + "22" : COLORS.accent2 + "22",
            color: t.type === "تست" ? COLORS.accent : COLORS.accent2,
          }}>{t.type}</span>
        )}
        {t.rangeFrom != null && t.rangeTo != null ? (
          <span style={{
            fontSize: 10.5, fontWeight: 600, padding: "2px 8px", borderRadius: 999,
            background: COLORS.surface2, color: COLORS.accentSoft,
          }}>
            از {toFa(t.rangeFrom)} تا {toFa(t.rangeTo)} · ضریب {toFa(t.rangeStep || 1)}
          </span>
        ) : null}
        {t.targetCount ? (
          <span style={{ fontSize: 10.5, color: COLORS.muted }}>
            {toFa(t.doneCount || 0)} از {toFa(t.targetCount)} · باقی {toFa(remaining)}
          </span>
        ) : t.doneCount ? (
          <span style={{ fontSize: 10.5, color: COLORS.muted }}>{toFa(t.doneCount)} {t.type || "تمرین"} انجام‌شده</span>
        ) : null}
        {!t.done && (
          <button onClick={() => setProgOpen((v) => !v)} className="kn-btn" style={{ background: "none", border: "none", color: COLORS.accent, fontSize: 10.5, fontWeight: 700 }}>
            + ثبت پیشرفت
          </button>
        )}
        {t.done && (
          <button onClick={onToggle} className="kn-btn" style={{ background: "none", border: "none", color: COLORS.accentSoft, fontSize: 10.5, fontWeight: 700 }}>
            بازگرداندن به انجام‌نشده
          </button>
        )}
      </div>
      {progOpen && !t.done && (
        <div className="kn-fade" style={{ display: "flex", gap: 6, marginTop: 8, marginRight: 24 }}>
          <input
            type="number" min="1" value={progVal} onChange={(e) => setProgVal(e.target.value)} placeholder="تعداد"
            style={{ flex: 1, background: COLORS.surface2, border: `1px solid ${COLORS.border}`, borderRadius: 8, padding: "6px 10px", color: COLORS.text, fontSize: 12, outline: "none" }}
          />
          <button onClick={() => { onAddProgress(parseInt(progVal, 10) || 0); setProgVal(""); setProgOpen(false); }} className="kn-btn" style={{
            background: COLORS.accent2, border: "none", borderRadius: 8, padding: "0 12px", color: "#062420", fontSize: 12, fontWeight: 700,
          }}>ثبت</button>
        </div>
      )}
    </div>
  );
}






function normalizeExam(ex) {
  if (!ex) return null;
  if (Array.isArray(ex.subjects) && ex.subjects.length) return ex;
  // سازگاری با آزمون‌های تک‌درسی قدیمی
  if (ex.subjectId) {
    return {
      ...ex,
      subjects: [{
        subjectId: ex.subjectId,
        topic: ex.topic || "",
        scoreType: ex.scoreType || "percent",
        score: ex.score,
        maxScore: ex.maxScore,
        coefficient: 1,
      }],
    };
  }
  return { ...ex, subjects: [] };
}

function subjectPercent(row, exam) {
  if (!row) return 0;
  if (row.scoreType === "raw") {
    const maxS = Number(row.maxScore) || 20;
    if (maxS <= 0) return 0;
    if (row.score == null || row.score === "") return 0;
    return Math.max(0, Math.min(100, ((Number(row.score) || 0) / maxS) * 100));
  }
  if (row.score == null || row.score === "" || (typeof row.score === "number" && isNaN(row.score))) {
    if (exam && exam.questionCount && isPercentExam(exam)) {
      const st = examQuestionStats(exam, row.subjectId);
      const p = percentFromAnswers(st.correct, st.wrong, st.total || exam.questionCount);
      if (p != null) return p;
    }
    return 0;
  }
  if (row.scoreType === "percent") return Math.max(0, Math.min(100, Number(row.score) || 0));
  const maxS = Number(row.maxScore) || 20;
  if (maxS <= 0) return 0;
  return Math.max(0, Math.min(100, ((Number(row.score) || 0) / maxS) * 100));
}

function examWeightedPercent(ex) {
  const n = normalizeExam(ex);
  const rows = n?.subjects || [];
  if (!rows.length) return 0;
  let wSum = 0, pSum = 0;
  rows.forEach((r) => {
    const c = Math.max(0, Number(r.coefficient) || 0);
    if (c <= 0) return;
    wSum += c;
    pSum += subjectPercent(r, n) * c;
  });
  if (wSum <= 0) {
    const avg = rows.reduce((a, r) => a + subjectPercent(r, n), 0) / rows.length;
    return Math.round(avg * 10) / 10;
  }
  return Math.round((pSum / wSum) * 10) / 10;
}

function examSubjectIds(ex) {
  const n = normalizeExam(ex);
  return (n?.subjects || []).map((r) => r.subjectId).filter(Boolean);
}

function studyHoursForSubjectBeforeExam(subjectId, exam, allExams, sessions) {
  const same = (allExams || [])
    .map(normalizeExam)
    .filter((e) => e && e.id !== exam.id && e.date <= exam.date && examSubjectIds(e).includes(subjectId))
    .sort((a, b) => b.date.localeCompare(a.date) || (b.createdAt || 0) - (a.createdAt || 0));
  const prevDate = same.length ? same[0].date : null;
  const mins = (sessions || [])
    .filter((s) => s.subjectId === subjectId && s.date <= exam.date && (!prevDate || s.date > prevDate))
    .reduce((a, s) => a + (Number(s.minutes) || 0), 0);
  return Math.round((mins / 60) * 10) / 10;
}

function studyHoursBeforeExam(exam, allExams, sessions) {
  const ids = examSubjectIds(exam);
  if (!ids.length) return 0;
  const total = ids.reduce((a, id) => a + studyHoursForSubjectBeforeExam(id, exam, allExams, sessions), 0);
  return Math.round(total * 10) / 10;
}

function ExamTrendChart({ points }) {
  if (!points || points.length === 0) {
    return <div style={{ color: COLORS.muted, fontSize: 12, textAlign: "center", padding: 20 }}>برای نمودار حداقل یک آزمون ثبت کن</div>;
  }
  const W = 320, H = 160, padL = 32, padR = 12, padT = 16, padB = 28;
  const innerW = W - padL - padR;
  const innerH = H - padT - padB;
  const maxH = Math.max(1, ...points.map((p) => p.hours));
  const n = points.length;
  const xAt = (i) => padL + (n === 1 ? innerW / 2 : (i / (n - 1)) * innerW);
  const yPct = (v) => padT + innerH * (1 - Math.max(0, Math.min(100, v)) / 100);
  const yHrs = (v) => padT + innerH * (1 - Math.max(0, v) / maxH);

  const pctPath = points.map((p, i) => `${i === 0 ? "M" : "L"} ${xAt(i).toFixed(1)} ${yPct(p.percent).toFixed(1)}`).join(" ");
  const hrsPath = points.map((p, i) => `${i === 0 ? "M" : "L"} ${xAt(i).toFixed(1)} ${yHrs(p.hours).toFixed(1)}`).join(" ");

  return (
    <div style={{ width: "100%", overflowX: "auto" }}>
      <svg viewBox={`0 0 ${W} ${H}`} width="100%" style={{ maxWidth: 420, display: "block", margin: "0 auto" }}>
        {[0, 25, 50, 75, 100].map((g) => (
          <g key={g}>
            <line x1={padL} y1={yPct(g)} x2={W - padR} y2={yPct(g)} stroke={COLORS.border} strokeWidth="1" />
            <text x={padL - 4} y={yPct(g) + 3} textAnchor="end" fontSize="9" fill={COLORS.muted}>{toFa(g)}</text>
          </g>
        ))}
        <path d={hrsPath} fill="none" stroke={COLORS.accent2} strokeWidth="2" strokeLinejoin="round" strokeLinecap="round" />
        <path d={pctPath} fill="none" stroke={COLORS.accent} strokeWidth="2.5" strokeLinejoin="round" strokeLinecap="round" />
        {points.map((p, i) => (
          <g key={i}>
            <circle cx={xAt(i)} cy={yHrs(p.hours)} r="3.5" fill={COLORS.accent2} />
            <circle cx={xAt(i)} cy={yPct(p.percent)} r="4" fill={COLORS.accent} />
            <text x={xAt(i)} y={H - 8} textAnchor="middle" fontSize="9" fill={COLORS.muted}>{p.label}</text>
          </g>
        ))}
      </svg>
      <div style={{ display: "flex", justifyContent: "center", gap: 16, marginTop: 6, fontSize: 11, color: COLORS.muted }}>
        <span><span style={{ display: "inline-block", width: 10, height: 3, background: COLORS.accent, marginLeft: 4, verticalAlign: "middle" }} />نمره/درصد</span>
        <span><span style={{ display: "inline-block", width: 10, height: 3, background: COLORS.accent2, marginLeft: 4, verticalAlign: "middle" }} />ساعت مطالعه</span>
      </div>
    </div>
  );
}

/** نمودار یک آزمون: برای هر درس همان آزمون */
function ExamSubjectsChart({ exam, allExams, sessions }) {
  const nExam = normalizeExam(exam);
  const rows = nExam?.subjects || [];
  if (!rows.length) {
    return <div style={{ color: COLORS.muted, fontSize: 11.5, textAlign: "center", padding: 12 }}>درسی ثبت نشده</div>;
  }
  const points = rows.map((row) => {
    const sub = curriculumById(row.subjectId);
    return {
      name: sub?.name || "—",
      color: sub?.color || COLORS.accent,
      percent: Math.round(subjectPercent(row, exam) * 10) / 10,
      hours: studyHoursForSubjectBeforeExam(row.subjectId, nExam, allExams, sessions),
      coef: Number(row.coefficient) || 1,
    };
  });
  const maxH = Math.max(1, ...points.map((p) => p.hours));
  const barW = Math.max(28, Math.min(48, Math.floor(280 / points.length)));

  return (
    <div style={{ marginTop: 10, paddingTop: 10, borderTop: `1px solid ${COLORS.border}` }}>
      <div style={{ fontSize: 11.5, fontWeight: 700, color: COLORS.muted, marginBottom: 8 }}>نمودار این آزمون (درس‌ها)</div>
      <div style={{ display: "flex", alignItems: "flex-end", gap: 8, minHeight: 120, overflowX: "auto", paddingBottom: 4 }}>
        {points.map((p, i) => {
          const hPct = Math.max(4, Math.round((p.percent / 100) * 90));
          const hHrs = Math.max(p.hours > 0 ? 4 : 0, Math.round((p.hours / maxH) * 90));
          return (
            <div key={i} style={{ flex: "0 0 auto", width: barW + 16, textAlign: "center" }}>
              <div style={{ display: "flex", alignItems: "flex-end", justifyContent: "center", gap: 3, height: 100 }}>
                <div title={`درصد: ${p.percent}`} style={{
                  width: 10, height: hPct, background: COLORS.accent, borderRadius: "4px 4px 0 0",
                }} />
                <div title={`ساعت: ${p.hours}`} style={{
                  width: 10, height: hHrs, background: COLORS.accent2, borderRadius: "4px 4px 0 0",
                }} />
              </div>
              <div style={{ fontSize: 9.5, color: COLORS.muted, marginTop: 4, lineHeight: 1.3, maxWidth: barW + 16, overflow: "hidden", textOverflow: "ellipsis", whiteSpace: "nowrap" }}>
                {p.name}
              </div>
              <div style={{ fontSize: 9, color: COLORS.accentSoft, marginTop: 2 }}>{toFa(p.percent)}٪</div>
              <div style={{ fontSize: 9, color: COLORS.muted }}>{toFa(p.hours)} س</div>
            </div>
          );
        })}
      </div>
      <div style={{ display: "flex", gap: 12, marginTop: 6, fontSize: 10.5, color: COLORS.muted }}>
        <span><span style={{ display: "inline-block", width: 8, height: 8, background: COLORS.accent, borderRadius: 2, marginLeft: 4 }} />درصد</span>
        <span><span style={{ display: "inline-block", width: 8, height: 8, background: COLORS.accent2, borderRadius: 2, marginLeft: 4 }} />ساعت مطالعه</span>
      </div>
    </div>
  );
}

function emptySubjectRow() {
  return {
    key: Date.now().toString(36) + Math.random().toString(36).slice(2, 5),
    subjectId: CURRICULUM[0]?.id || "",
    topic: "",
    scoreType: "percent",
    score: "",
    maxScore: "20",
    coefficient: "1",
  };
}




function ExamDetailView({ exam, onBack, onUpdate }) {
  const [questionCount, setQuestionCount] = useState(String(exam.questionCount || ""));
  const [answers, setAnswers] = useState(() => ({ ...(exam.answers || {}) }));
  const [starred, setStarred] = useState(() => ({ ...(exam.starred || {}) }));
  const [qMeta, setQMeta] = useState(() => ({ ...(exam.qMeta || {}) }));
  const [msg, setMsg] = useState("");
  const [topicPickQ, setTopicPickQ] = useState(null); // question number
  const [pickSubject, setPickSubject] = useState("");
  const [pickChapter, setPickChapter] = useState(null);

  const count = parseInt(questionCount, 10) || 0;

  function persist(patch) {
    onUpdate({ ...exam, ...patch });
  }

  function saveQuestionsSetup() {
    const n = parseInt(questionCount, 10);
    if (!n || n < 1 || n > 500) {
      setMsg("تعداد سوال باید بین ۱ تا ۵۰۰ باشد");
      return;
    }
    const patch = { questionCount: n, answers, starred, qMeta };
    let computed = null;
    if (isPercentExam(exam)) {
      const stats = examQuestionStats({ ...exam, ...patch });
      computed = percentFromAnswers(stats.correct, stats.wrong, stats.total || n);
      let subjects = exam.subjects || [];
      if (computed != null && subjects.length) {
        subjects = subjects.map((s) => {
          if (s.scoreType === "raw") return s;
          if (s.score != null && s.score !== "") return s;
          const st = examQuestionStats({ ...exam, ...patch }, s.subjectId);
          const p = percentFromAnswers(st.correct, st.wrong, st.total || n);
          return { ...s, scoreType: "percent", score: p != null ? p : s.score, fromAnswers: true };
        });
        patch.subjects = subjects;
      }
    }
    persist(patch);
    setMsg(computed != null ? `ذخیره شد · درصد محاسبه‌شده: ${toFa(computed)}٪` : "ذخیره شد");
  }

  function setStatus(q, status) {
    setAnswers((prev) => {
      const next = { ...prev, [q]: prev[q] === status ? "" : status };
      persist({ answers: next, starred, qMeta, questionCount: count || exam.questionCount });
      return next;
    });
  }

  function toggleStar(q) {
    setStarred((prev) => {
      const on = !prev[q];
      const next = { ...prev, [q]: on };
      let nextMeta = { ...qMeta };
      if (on && !nextMeta[q]) {
        nextMeta[q] = {
          subjectId: exam.subjects?.[0]?.subjectId || CURRICULUM[0]?.id || "",
          chapterNum: null,
          topic: "",
        };
        setQMeta(nextMeta);
      }
      persist({ answers, starred: next, qMeta: nextMeta, questionCount: count || exam.questionCount });
      return next;
    });
  }

  function openTopicPick(q) {
    const meta = qMeta[q] || {};
    setTopicPickQ(q);
    setPickSubject(meta.subjectId || exam.subjects?.[0]?.subjectId || CURRICULUM[0]?.id || "");
    setPickChapter(meta.chapterNum != null ? meta.chapterNum : null);
  }

  function saveTopicPick() {
    if (topicPickQ == null) return;
    const ch = chaptersOf(pickSubject).find((x) => x.num === Number(pickChapter));
    const topic = ch ? ch.title : "";
    setQMeta((prev) => {
      const next = {
        ...prev,
        [topicPickQ]: {
          subjectId: pickSubject,
          chapterNum: pickChapter,
          topic,
        },
      };
      persist({ answers, starred, qMeta: next, questionCount: count || exam.questionCount });
      return next;
    });
    setTopicPickQ(null);
    setMsg(`مبحث سوال ${toFa(topicPickQ)} ذخیره شد`);
  }

  const percentMode = isPercentExam(exam);

  const stats = useMemo(() => {
    let c = 0, w = 0, b = 0, s = 0;
    for (let i = 1; i <= count; i++) {
      const st = answers[i];
      if (st === "correct") c++;
      else if (st === "wrong") w++;
      else if (st === "blank") b++;
      if (starred[i]) s++;
    }
    const pct = percentMode ? percentFromAnswers(c, w, count) : null;
    const pctNoWrong = percentMode ? percentIfNoWrong(c, count) : null;
    return { c, w, b, s, pct, pctNoWrong };
  }, [answers, starred, count, percentMode]);

  return (
    <div className="kn-fade">
      <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 12 }}>
        <button onClick={onBack} className="kn-btn" style={{ background: "none", border: "none", color: COLORS.accent, fontWeight: 700, fontSize: 13 }}>← بازگشت</button>
        <div style={{ fontSize: 14, fontWeight: 800 }}>تحلیل آزمون</div>
        <div style={{ width: 50 }} />
      </div>
      <div style={{ fontSize: 13, color: COLORS.muted, marginBottom: 12 }}>
        {exam.title || "آزمون"} · {toJalaliShort(new Date(exam.date + "T00:00:00"))}
      </div>

      <div style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14, padding: 14, marginBottom: 14 }}>
        <div style={{ fontSize: 13, fontWeight: 700, marginBottom: 8 }}>تعداد سوالات</div>
        <div style={{ display: "flex", gap: 8 }}>
          <input type="number" min="1" max="500" value={questionCount} onChange={(e) => setQuestionCount(e.target.value)}
            placeholder="مثلاً ۵۰"
            style={{ flex: 1, background: COLORS.surface2, border: `1px solid ${COLORS.border}`, borderRadius: 10, padding: "9px 12px", color: COLORS.text, fontSize: 13, outline: "none" }} />
          <button onClick={saveQuestionsSetup} className="kn-btn" style={{ background: COLORS.accent, color: "#1a1400", border: "none", borderRadius: 10, padding: "0 16px", fontWeight: 700 }}>اعمال</button>
        </div>
        {count > 0 && (
          <div style={{ display: "flex", flexWrap: "wrap", gap: 8, marginTop: 10, fontSize: 11.5 }}>
            <span style={{ color: "#22c55e" }}>درست {toFa(stats.c)}</span>
            <span style={{ color: COLORS.danger }}>غلط {toFa(stats.w)}</span>
            <span style={{ color: COLORS.muted }}>نزده {toFa(stats.b)}</span>
            <span style={{ color: COLORS.accent }}>⭐ {toFa(stats.s)}</span>
            {percentMode && stats.pct != null && (
              <span style={{ color: COLORS.accentSoft, fontWeight: 800 }}>درصد: {toFa(stats.pct)}٪</span>
            )}
            {percentMode && stats.pctNoWrong != null && (
              <span style={{ color: "#38bdf8", fontWeight: 800 }}>بدون غلط: {toFa(stats.pctNoWrong)}٪</span>
            )}
          </div>
        )}
        <div style={{ fontSize: 10.5, color: COLORS.muted, marginTop: 8, lineHeight: 1.6 }}>
          {percentMode
            ? "آزمون درصدی · فرمول: (درست × ۳ − غلط) ÷ (کل × ۳) × ۱۰۰ · «بدون غلط» یعنی اگر غلط‌ها صفر بودند."
            : "این آزمون نمره‌ای ثبت شده؛ درصد از روی درست/غلط حساب نمی‌شود."}
        </div>
      </div>

      {count > 0 && (
        <div style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14, padding: 14, marginBottom: 14 }}>
          <div style={{ fontSize: 12, color: COLORS.muted, marginBottom: 10, lineHeight: 1.7 }}>
            روی <b style={{ color: COLORS.text }}>شماره سوال</b> بزن تا درس و فصل/مبحث را انتخاب کنی.
          </div>
          <div style={{ fontSize: 13, fontWeight: 700, marginBottom: 10 }}>وضعیت هر سوال</div>
          <div style={{ display: "flex", flexDirection: "column", gap: 6, maxHeight: 480, overflowY: "auto" }}>
            {Array.from({ length: count }, (_, i) => i + 1).map((q) => {
              const st = answers[q];
              const isStar = !!starred[q];
              const meta = qMeta[q];
              const sub = meta?.subjectId ? curriculumById(meta.subjectId) : null;
              return (
                <div key={q} style={{
                  background: COLORS.bg, borderRadius: 12, padding: "8px 10px",
                  border: `1px solid ${COLORS.border}`,
                }}>
                  <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
                    <button type="button" onClick={() => openTopicPick(q)} className="kn-btn" style={{
                      minWidth: 40, background: meta?.topic ? COLORS.accent + "22" : COLORS.surface2,
                      border: `1px solid ${meta?.topic ? COLORS.accent : COLORS.border}`,
                      borderRadius: 10, padding: "6px 8px", color: meta?.topic ? COLORS.accentSoft : COLORS.text,
                      fontWeight: 800, fontSize: 13,
                    }}>{toFa(q)}</button>
                    <button type="button" onClick={() => setStatus(q, "correct")} title="درست" className="kn-btn" style={{
                      width: 36, height: 36, borderRadius: 10, display: "flex", alignItems: "center", justifyContent: "center",
                      background: st === "correct" ? "#22c55e28" : COLORS.surface2,
                      border: `1.5px solid ${st === "correct" ? "#22c55e" : COLORS.border}`,
                      color: st === "correct" ? "#22c55e" : COLORS.muted, padding: 0,
                    }}>
                      <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2.5" strokeLinecap="round" strokeLinejoin="round">
                        <circle cx="12" cy="12" r="10" fill={st === "correct" ? "currentColor" : "none"} opacity={st === "correct" ? 0.2 : 1} />
                        <path d="m8.5 12.5 2.5 2.5 4.5-5" />
                      </svg>
                    </button>
                    <button type="button" onClick={() => setStatus(q, "wrong")} title="غلط" className="kn-btn" style={{
                      width: 36, height: 36, borderRadius: 10, display: "flex", alignItems: "center", justifyContent: "center",
                      background: st === "wrong" ? COLORS.danger + "28" : COLORS.surface2,
                      border: `1.5px solid ${st === "wrong" ? COLORS.danger : COLORS.border}`,
                      color: st === "wrong" ? COLORS.danger : COLORS.muted, padding: 0,
                    }}>
                      <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2.5" strokeLinecap="round" strokeLinejoin="round">
                        <circle cx="12" cy="12" r="10" fill={st === "wrong" ? "currentColor" : "none"} opacity={st === "wrong" ? 0.2 : 1} />
                        <path d="m9 9 6 6" /><path d="m15 9-6 6" />
                      </svg>
                    </button>
                    <button type="button" onClick={() => setStatus(q, "blank")} title="نزده" className="kn-btn" style={{
                      width: 36, height: 36, borderRadius: 10, display: "flex", alignItems: "center", justifyContent: "center",
                      background: st === "blank" ? "#94a3b828" : COLORS.surface2,
                      border: `1.5px solid ${st === "blank" ? "#94a3b8" : COLORS.border}`,
                      color: st === "blank" ? "#cbd5e1" : COLORS.muted, padding: 0,
                    }}>
                      <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2.5" strokeLinecap="round">
                        <circle cx="12" cy="12" r="10" />
                        <path d="M8 12h8" />
                      </svg>
                    </button>
                    <button type="button" onClick={() => toggleStar(q)} title="مهم" className="kn-btn" style={{
                      width: 36, height: 36, borderRadius: 10, display: "flex", alignItems: "center", justifyContent: "center",
                      background: isStar ? COLORS.accent + "28" : COLORS.surface2,
                      border: `1.5px solid ${isStar ? COLORS.accent : COLORS.border}`,
                      color: isStar ? COLORS.accent : COLORS.muted, padding: 0, marginRight: "auto",
                    }}>
                      <svg width="18" height="18" viewBox="0 0 24 24" fill={isStar ? "currentColor" : "none"} stroke="currentColor" strokeWidth="2" strokeLinejoin="round">
                        <polygon points="12 2 15.09 8.26 22 9.27 17 14.14 18.18 21.02 12 17.77 5.82 21.02 7 14.14 2 9.27 8.91 8.26 12 2" />
                      </svg>
                    </button>
                  </div>
                  {meta?.topic && (
                    <div style={{ fontSize: 11, color: COLORS.muted, marginTop: 6, paddingRight: 4 }}>
                      {sub?.name || ""} · {meta.topic}
                    </div>
                  )}
                </div>
              );
            })}
          </div>
        </div>
      )}

      {topicPickQ != null && (
        <div style={{
          position: "fixed", inset: 0, background: "rgba(0,0,0,0.55)", zIndex: 80,
          display: "flex", alignItems: "flex-end", justifyContent: "center",
        }} onClick={() => setTopicPickQ(null)}>
          <div onClick={(e) => e.stopPropagation()} style={{
            width: "100%", maxWidth: 480, maxHeight: "80vh", overflowY: "auto",
            background: COLORS.surface, borderRadius: "18px 18px 0 0", padding: 16,
            border: `1px solid ${COLORS.border}`,
          }}>
            <div style={{ fontSize: 15, fontWeight: 800, marginBottom: 4 }}>مبحث سوال {toFa(topicPickQ)}</div>
            <div style={{ fontSize: 12, color: COLORS.muted, marginBottom: 12 }}>اول درس، بعد فصل را انتخاب کن</div>

            <div style={{ fontSize: 11, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>درس</div>
            <div style={{ display: "flex", flexWrap: "wrap", gap: 6, marginBottom: 14 }}>
              {CURRICULUM.filter((s) => s.groupLabel !== "ویژه").map((s) => (
                <button key={s.id} type="button" onClick={() => { setPickSubject(s.id); setPickChapter(null); }} className="kn-btn" style={{
                  border: `1px solid ${pickSubject === s.id ? s.color : COLORS.border}`,
                  background: pickSubject === s.id ? s.color + "22" : COLORS.bg,
                  color: pickSubject === s.id ? s.color : COLORS.muted,
                  borderRadius: 999, padding: "5px 10px", fontSize: 11, fontWeight: 700,
                }}>{s.name}</button>
              ))}
            </div>

            <div style={{ fontSize: 11, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>
              {isLessonBasedSubject(pickSubject) ? "درس" : "فصل"}
            </div>
            <div style={{ display: "flex", flexDirection: "column", gap: 6, marginBottom: 14 }}>
              {chaptersOf(pickSubject).map((ch) => (
                <button key={ch.num} type="button" onClick={() => setPickChapter(ch.num)} className="kn-btn" style={{
                  textAlign: "right",
                  border: `1.5px solid ${pickChapter === ch.num ? COLORS.accent : COLORS.border}`,
                  background: pickChapter === ch.num ? COLORS.accent + "18" : COLORS.bg,
                  color: pickChapter === ch.num ? COLORS.accentSoft : COLORS.text,
                  borderRadius: 10, padding: "10px 12px", fontSize: 12.5, fontWeight: 600,
                }}>
                  {isLessonBasedSubject(pickSubject)
                    ? ch.title
                    : (<><span style={{ color: COLORS.muted, marginLeft: 6 }}>فصل {toFa(ch.num)} — </span>{ch.title}</>)}
                </button>
              ))}
            </div>

            <div style={{ display: "flex", gap: 8 }}>
              <button onClick={saveTopicPick} className="kn-btn" style={{
                flex: 1, background: COLORS.accent, color: "#1a1400", border: "none", borderRadius: 10, padding: "11px 0", fontWeight: 800,
              }}>ذخیره مبحث</button>
              <button onClick={() => setTopicPickQ(null)} className="kn-btn" style={{
                flex: 1, background: "none", border: `1px solid ${COLORS.border}`, borderRadius: 10, padding: "11px 0", color: COLORS.muted,
              }}>انصراف</button>
            </div>
          </div>
        </div>
      )}

      {msg && <div style={{ fontSize: 12, color: COLORS.accentSoft, textAlign: "center", marginBottom: 12 }}>{msg}</div>}
    </div>
  );
}

function ExamsView({ exams, setExams, sessions, profile }) {
  const [section, setSection] = useState("results");
  const [detailId, setDetailId] = useState(null);
  const [title, setTitle] = useState("");
  const [rows, setRows] = useState([emptySubjectRow()]);
  const [date, setDate] = useState(todayKey());
  const [note, setNote] = useState("");
  const [filterSubject, setFilterSubject] = useState("all");
  const [msg, setMsg] = useState("");
  const [impSubject, setImpSubject] = useState("");
  const [impTopic, setImpTopic] = useState("");

  const sorted = useMemo(() => {
    return (exams || []).map(normalizeExam).filter(Boolean)
      .sort((a, b) => a.date.localeCompare(b.date) || (a.createdAt || 0) - (b.createdAt || 0));
  }, [exams]);

  const filtered = useMemo(() => {
    if (filterSubject === "all") return sorted;
    return sorted.filter((e) => examSubjectIds(e).includes(filterSubject));
  }, [sorted, filterSubject]);

  const importantItems = useMemo(() => {
    const list = [];
    (exams || []).map(normalizeExam).forEach((ex) => {
      const starred = ex.starred || {};
      const qMeta = ex.qMeta || {};
      Object.keys(starred).forEach((k) => {
        if (!starred[k]) return;
        const q = parseInt(k, 10);
        const meta = qMeta[k] || {};
        list.push({
          examId: ex.id,
          examTitle: ex.title || "آزمون",
          examDate: ex.date,
          q,
          subjectId: meta.subjectId || ex.subjects?.[0]?.subjectId || "",
          topic: meta.topic || "",
          status: (ex.answers || {})[k] || "",
        });
      });
    });
    return list;
  }, [exams]);

  const impSubjects = useMemo(() => {
    const ids = [...new Set(importantItems.map((x) => x.subjectId).filter(Boolean))];
    return ids.map((id) => curriculumById(id)).filter(Boolean);
  }, [importantItems]);

  const impTopics = useMemo(() => {
    if (!impSubject) return [];
    return [...new Set(importantItems.filter((x) => x.subjectId === impSubject).map((x) => x.topic || "").filter(Boolean))];
  }, [importantItems, impSubject]);

  const filteredImportant = useMemo(() => {
    return importantItems.filter((x) => {
      if (impSubject && x.subjectId !== impSubject) return false;
      if (impTopic && (x.topic || "") !== impTopic) return false;
      return true;
    }).sort((a, b) => (b.examDate || "").localeCompare(a.examDate || "") || a.q - b.q);
  }, [importantItems, impSubject, impTopic]);

  function updateRow(key, patch) {
    setRows((prev) => prev.map((r) => (r.key === key ? { ...r, ...patch } : r)));
  }
  function addRow() { setRows((prev) => [...prev, emptySubjectRow()]); }
  function removeRow(key) { setRows((prev) => (prev.length <= 1 ? prev : prev.filter((r) => r.key !== key))); }

  function addExam() {
    const parsed = [];
    for (const r of rows) {
      if (!r.subjectId) { setMsg("برای هر ردیف یک درس انتخاب کن"); return; }
      const rawScore = String(r.score ?? "").trim();
      const hasScore = rawScore !== "";
      const sc = hasScore ? parseFloat(rawScore) : null;
      if (hasScore && isNaN(sc)) { setMsg("نمره نامعتبر است"); return; }
      const coef = parseFloat(r.coefficient);
      if (isNaN(coef) || coef <= 0) { setMsg("ضریب هر درس باید عدد مثبت باشد"); return; }
      if (hasScore && r.scoreType === "percent" && (sc < 0 || sc > 100)) { setMsg("درصد باید بین ۰ تا ۱۰۰ باشد"); return; }
      let maxS = 20;
      if (hasScore && r.scoreType === "raw") {
        maxS = parseFloat(r.maxScore);
        if (isNaN(maxS) || maxS <= 0) { setMsg("سقف نمره نامعتبر است"); return; }
        if (sc < 0 || sc > maxS) { setMsg("نمره نباید از سقف بیشتر باشد"); return; }
      }
      parsed.push({
        subjectId: r.subjectId, topic: (r.topic || "").trim(), scoreType: r.scoreType,
        score: hasScore ? sc : null, maxScore: r.scoreType === "raw" ? maxS : null, coefficient: coef,
      });
    }
    const item = {
      id: Date.now().toString(36) + Math.random().toString(36).slice(2, 6),
      title: title.trim(), subjects: parsed, date: date || todayKey(), note: note.trim(), createdAt: Date.now(),
      questionCount: 0, answers: {}, starred: {}, qMeta: {},
    };
    setExams((prev) => [...(prev || []), item]);
    setTitle(""); setRows([emptySubjectRow()]); setNote(""); setMsg("آزمون ثبت شد");
  }

  function deleteExam(id) {
    if (!confirm("این نتیجه آزمون حذف شود؟")) return;
    setExams((prev) => (prev || []).filter((e) => e.id !== id));
  }

  function updateExam(next) {
    setExams((prev) => (prev || []).map((e) => (e.id === next.id ? next : e)));
  }

  const subjectsInExams = useMemo(() => {
    const ids = new Set();
    (exams || []).map(normalizeExam).forEach((e) => { examSubjectIds(e).forEach((id) => ids.add(id)); });
    return [...ids].map((id) => curriculumById(id)).filter(Boolean);
  }, [exams]);

  const detailExam = detailId ? normalizeExam((exams || []).find((e) => e.id === detailId)) : null;
  if (detailExam) {
    return (
      <ExamDetailView
        exam={detailExam}
        onBack={() => setDetailId(null)}
        onUpdate={updateExam}
      />
    );
  }

  return (
    <div className="kn-fade">
      <div style={{ display: "flex", gap: 6, marginBottom: 14 }}>
        <button onClick={() => setSection("results")} className="kn-btn" style={{
          flex: 1, border: `1.5px solid ${section === "results" ? COLORS.accent : COLORS.border}`,
          background: section === "results" ? COLORS.accent + "22" : "transparent",
          color: section === "results" ? COLORS.accentSoft : COLORS.muted,
          borderRadius: 10, padding: "9px 0", fontSize: 13, fontWeight: 700,
        }}>نتایج آزمون</button>
        <button onClick={() => setSection("important")} className="kn-btn" style={{
          flex: 1, border: `1.5px solid ${section === "important" ? COLORS.accent : COLORS.border}`,
          background: section === "important" ? COLORS.accent + "22" : "transparent",
          color: section === "important" ? COLORS.accentSoft : COLORS.muted,
          borderRadius: 10, padding: "9px 0", fontSize: 13, fontWeight: 700,
        }}>سوالات مهم</button>
      </div>

      {section === "important" ? (
        <div>
          <div style={{ fontSize: 12, color: COLORS.muted, marginBottom: 12, lineHeight: 1.7 }}>
            سوالاتی که با ستاره علامت زدی. اول درس و مبحث را انتخاب کن.
          </div>
          <div style={{ marginBottom: 10 }}>
            <div style={{ fontSize: 11, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>درس</div>
            <div style={{ display: "flex", flexWrap: "wrap", gap: 6 }}>
              <button type="button" onClick={() => { setImpSubject(""); setImpTopic(""); }} className="kn-btn" style={{
                border: `1px solid ${!impSubject ? COLORS.accent : COLORS.border}`,
                background: !impSubject ? COLORS.accent + "22" : "transparent",
                color: !impSubject ? COLORS.accentSoft : COLORS.muted,
                borderRadius: 999, padding: "4px 10px", fontSize: 11, fontWeight: 700,
              }}>همه</button>
              {impSubjects.map((s) => (
                <button key={s.id} type="button" onClick={() => { setImpSubject(s.id); setImpTopic(""); }} className="kn-btn" style={{
                  border: `1px solid ${impSubject === s.id ? s.color : COLORS.border}`,
                  background: impSubject === s.id ? s.color + "22" : "transparent",
                  color: impSubject === s.id ? s.color : COLORS.muted,
                  borderRadius: 999, padding: "4px 10px", fontSize: 11, fontWeight: 700,
                }}>{s.name}</button>
              ))}
            </div>
          </div>
          {impTopics.length > 0 && (
            <div style={{ marginBottom: 12 }}>
              <div style={{ fontSize: 11, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>مبحث</div>
              <div style={{ display: "flex", flexWrap: "wrap", gap: 6 }}>
                <button type="button" onClick={() => setImpTopic("")} className="kn-btn" style={{
                  border: `1px solid ${!impTopic ? COLORS.accent : COLORS.border}`,
                  background: !impTopic ? COLORS.accent + "22" : "transparent",
                  color: !impTopic ? COLORS.accentSoft : COLORS.muted,
                  borderRadius: 999, padding: "4px 10px", fontSize: 11, fontWeight: 700,
                }}>همه مباحث</button>
                {impTopics.map((t) => (
                  <button key={t} type="button" onClick={() => setImpTopic(t)} className="kn-btn" style={{
                    border: `1px solid ${impTopic === t ? COLORS.accent : COLORS.border}`,
                    background: impTopic === t ? COLORS.accent + "22" : "transparent",
                    color: impTopic === t ? COLORS.accentSoft : COLORS.muted,
                    borderRadius: 999, padding: "4px 10px", fontSize: 11, fontWeight: 700,
                  }}>{t}</button>
                ))}
              </div>
            </div>
          )}
          {filteredImportant.length === 0 ? (
            <div style={{ textAlign: "center", color: COLORS.muted, padding: 30, background: COLORS.surface, borderRadius: 12 }}>
              سوال مهمی ثبت نشده. از «تحلیل آزمون» کنار هر آزمون ستاره بگذار.
            </div>
          ) : (
            <div style={{ display: "flex", flexDirection: "column", gap: 8 }}>
              {filteredImportant.map((it) => {
                const sub = curriculumById(it.subjectId);
                return (
                  <div key={it.examId + "-" + it.q} style={{
                    background: COLORS.surface, border: `1px solid ${COLORS.border}`,
                    borderRadius: 12, padding: "12px 14px",
                  }}>
                    <div style={{ display: "flex", justifyContent: "space-between", gap: 8 }}>
                      <div style={{ fontSize: 13, fontWeight: 800 }}>سوال {toFa(it.q)}</div>
                      <Star size={14} color={COLORS.accent} />
                    </div>
                    <div style={{ fontSize: 11.5, color: COLORS.muted, marginTop: 4 }}>
                      {it.examTitle} · {toJalaliShort(new Date(it.examDate + "T00:00:00"))}
                    </div>
                    <div style={{ fontSize: 11.5, marginTop: 4 }}>
                      {sub?.name || "—"}{it.topic ? ` · ${it.topic}` : ""}
                      {it.status ? ` · ${it.status === "correct" ? "درست" : it.status === "wrong" ? "غلط" : "نزده"}` : ""}
                    </div>
                  </div>
                );
              })}
            </div>
          )}
        </div>
      ) : (
        <>
      <div style={{ fontSize: 15, fontWeight: 800, marginBottom: 4 }}>نتایج آزمون</div>
      <div style={{ fontSize: 12, color: COLORS.muted, marginBottom: 14, lineHeight: 1.7 }}>
        چند درس با ضریب جدا ثبت کن. نمره را می‌توانی خالی بگذاری و بعداً از روی درست/غلط در «تحلیل آزمون» حساب شود.
      </div>

      <div style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14, padding: 14, marginBottom: 16 }}>
        <input value={title} onChange={(e) => setTitle(e.target.value)} placeholder="عنوان آزمون (اختیاری، مثلاً آزمون جامع مرحله ۲)"
          style={{ width: "100%", background: COLORS.surface2, border: `1px solid ${COLORS.border}`, borderRadius: 10, padding: "8px 12px", color: COLORS.text, fontSize: 12.5, outline: "none", marginBottom: 12 }} />

        {rows.map((r, idx) => {
          const sub = curriculumById(r.subjectId);
          return (
            <div key={r.key} style={{
              border: `1px solid ${COLORS.border}`, borderRadius: 12, padding: 12, marginBottom: 10, background: COLORS.bg,
            }}>
              <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 8 }}>
                <div style={{ fontSize: 12.5, fontWeight: 700 }}>درس {toFa(idx + 1)}</div>
                {rows.length > 1 && (
                  <button type="button" onClick={() => removeRow(r.key)} className="kn-btn" style={{
                    background: "none", border: "none", color: COLORS.danger, fontSize: 11.5, fontWeight: 700,
                  }}>حذف درس</button>
                )}
              </div>
              <div style={{ marginBottom: 8 }}>
                <GroupedSubjectPicker items={CURRICULUM} mode="single" value={r.subjectId} onChange={(id) => updateRow(r.key, { subjectId: id })} />
              </div>
              <input value={r.topic} onChange={(e) => updateRow(r.key, { topic: e.target.value })} placeholder="مبحث این درس (اختیاری)"
                style={{ width: "100%", background: COLORS.surface2, border: `1px solid ${COLORS.border}`, borderRadius: 8, padding: "7px 10px", color: COLORS.text, fontSize: 12, outline: "none", marginBottom: 8 }} />
              <div style={{ display: "flex", gap: 6, marginBottom: 8 }}>
                <button type="button" onClick={() => updateRow(r.key, { scoreType: "percent" })} className="kn-btn" style={{
                  flex: 1, border: `1.5px solid ${r.scoreType === "percent" ? COLORS.accent : COLORS.border}`,
                  background: r.scoreType === "percent" ? COLORS.accent + "22" : "transparent",
                  color: r.scoreType === "percent" ? COLORS.accentSoft : COLORS.muted,
                  borderRadius: 8, padding: "6px 0", fontSize: 11.5, fontWeight: 700,
                }}>درصدی</button>
                <button type="button" onClick={() => updateRow(r.key, { scoreType: "raw" })} className="kn-btn" style={{
                  flex: 1, border: `1.5px solid ${r.scoreType === "raw" ? COLORS.accent : COLORS.border}`,
                  background: r.scoreType === "raw" ? COLORS.accent + "22" : "transparent",
                  color: r.scoreType === "raw" ? COLORS.accentSoft : COLORS.muted,
                  borderRadius: 8, padding: "6px 0", fontSize: 11.5, fontWeight: 700,
                }}>نمره خام</button>
              </div>
              <div style={{ display: "flex", gap: 8, marginBottom: 4 }}>
                <div style={{ flex: 1 }}>
                  <div style={{ fontSize: 10, color: COLORS.muted, marginBottom: 3 }}>{r.scoreType === "percent" ? "درصد" : "نمره"}</div>
                  <input type="number" value={r.score} onChange={(e) => updateRow(r.key, { score: e.target.value })}
                    placeholder={r.scoreType === "percent" ? "اختیاری — ۸۵" : "اختیاری — ۱۶"}
                    style={{ width: "100%", background: COLORS.surface2, border: `1px solid ${COLORS.border}`, borderRadius: 8, padding: "7px 8px", color: COLORS.text, fontSize: 12.5, outline: "none" }} />
                </div>
                {r.scoreType === "raw" && (
                  <div style={{ flex: 1 }}>
                    <div style={{ fontSize: 10, color: COLORS.muted, marginBottom: 3 }}>از چند؟</div>
                    <input type="number" value={r.maxScore} onChange={(e) => updateRow(r.key, { maxScore: e.target.value })} placeholder="۲۰"
                      style={{ width: "100%", background: COLORS.surface2, border: `1px solid ${COLORS.border}`, borderRadius: 8, padding: "7px 8px", color: COLORS.text, fontSize: 12.5, outline: "none" }} />
                  </div>
                )}
                <div style={{ flex: 0.9 }}>
                  <div style={{ fontSize: 10, color: COLORS.muted, marginBottom: 3 }}>ضریب</div>
                  <input type="number" min="0.1" step="0.5" value={r.coefficient} onChange={(e) => updateRow(r.key, { coefficient: e.target.value })} placeholder="۱"
                    style={{ width: "100%", background: COLORS.surface2, border: `1px solid ${COLORS.border}`, borderRadius: 8, padding: "7px 8px", color: COLORS.text, fontSize: 12.5, outline: "none" }} />
                </div>
              </div>
              {sub && (
                <div style={{ fontSize: 10.5, color: COLORS.muted, marginTop: 4 }}>
                  {sub.name} · ضریب {toFa(r.coefficient || 1)}
                </div>
              )}
            </div>
          );
        })}

        <button type="button" onClick={addRow} className="kn-btn" style={{
          width: "100%", background: COLORS.surface2, border: `1px dashed ${COLORS.border}`,
          borderRadius: 10, padding: "9px 0", color: COLORS.accentSoft, fontSize: 12.5, fontWeight: 700, marginBottom: 12,
        }}>+ افزودن درس دیگر</button>

        <div style={{ fontSize: 11, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>تاریخ آزمون</div>
        <div style={{ marginBottom: 10 }}>
          <JalaliDatePicker value={date} onChange={setDate} />
        </div>
        <input value={note} onChange={(e) => setNote(e.target.value)} placeholder="یادداشت کلی آزمون (اختیاری)"
          style={{ width: "100%", background: COLORS.surface2, border: `1px solid ${COLORS.border}`, borderRadius: 10, padding: "8px 12px", color: COLORS.text, fontSize: 12.5, outline: "none", marginBottom: 10 }} />
        <button onClick={addExam} className="kn-btn" style={{
          width: "100%", background: COLORS.accent, color: "#1a1400", border: "none", borderRadius: 10, padding: "11px 0", fontWeight: 800, fontSize: 13.5,
        }}>ثبت نتیجه آزمون</button>
        {msg && <div style={{ fontSize: 12, color: COLORS.accentSoft, textAlign: "center", marginTop: 8 }}>{msg}</div>}
      </div>

      {subjectsInExams.length > 0 && (
        <div style={{ display: "flex", gap: 6, flexWrap: "wrap", marginBottom: 12 }}>
          <button type="button" onClick={() => setFilterSubject("all")} className="kn-btn" style={{
            border: `1px solid ${filterSubject === "all" ? COLORS.accent : COLORS.border}`,
            background: filterSubject === "all" ? COLORS.accent + "22" : "transparent",
            color: filterSubject === "all" ? COLORS.accentSoft : COLORS.muted,
            borderRadius: 999, padding: "4px 10px", fontSize: 11, fontWeight: 700,
          }}>همه آزمون‌ها</button>
          {subjectsInExams.map((s) => (
            <button key={s.id} type="button" onClick={() => setFilterSubject(s.id)} className="kn-btn" style={{
              border: `1px solid ${filterSubject === s.id ? s.color : COLORS.border}`,
              background: filterSubject === s.id ? s.color + "22" : "transparent",
              color: filterSubject === s.id ? s.color : COLORS.muted,
              borderRadius: 999, padding: "4px 10px", fontSize: 11, fontWeight: 700,
            }}>{s.name}</button>
          ))}
        </div>
      )}

      <div style={{ fontSize: 13, fontWeight: 700, marginBottom: 8, color: COLORS.muted }}>آزمون‌های ثبت‌شده</div>
      {filtered.length === 0 ? (
        <div style={{ textAlign: "center", color: COLORS.muted, padding: 24, background: COLORS.surface, borderRadius: 12 }}>هنوز آزمونی ثبت نشده</div>
      ) : (
        <div style={{ display: "flex", flexDirection: "column", gap: 10 }}>
          {[...filtered].reverse().map((ex) => {
            const wAvg = examWeightedPercent(ex);
            const hours = studyHoursBeforeExam(ex, exams, sessions);
            const rowsEx = ex.subjects || [];
            const starCount = Object.values(ex.starred || {}).filter(Boolean).length;
            return (
              <div key={ex.id} style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 12, padding: "12px 14px" }}>
                <div style={{ display: "flex", justifyContent: "space-between", gap: 8, marginBottom: 6 }}>
                  <div>
                    <div style={{ fontSize: 13.5, fontWeight: 800 }}>
                      {ex.title || (rowsEx.length > 1 ? `آزمون چنددرسی (${toFa(rowsEx.length)} درس)` : (curriculumById(rowsEx[0]?.subjectId)?.name || "آزمون"))}
                    </div>
                    <div style={{ fontSize: 11, color: COLORS.muted, marginTop: 2 }}>
                      {toJalaliShort(new Date(ex.date + "T00:00:00"))}
                      {ex.questionCount ? ` · ${toFa(ex.questionCount)} سوال` : ""}
                      {starCount ? ` · ⭐ ${toFa(starCount)}` : ""}
                    </div>
                  </div>
                  <button onClick={() => deleteExam(ex.id)} className="kn-btn" style={{ background: "none", border: "none", color: COLORS.danger, fontSize: 11.5, fontWeight: 700 }}>حذف</button>
                </div>

                <div style={{ display: "flex", flexWrap: "wrap", gap: 8, marginBottom: 10, fontSize: 11.5 }}>
                  <span style={{ background: COLORS.accent + "22", color: COLORS.accentSoft, padding: "3px 10px", borderRadius: 999, fontWeight: 800 }}>
                    میانگین وزن‌دار: {toFa(wAvg)}٪
                  </span>
                  <span style={{ background: COLORS.surface2, color: COLORS.muted, padding: "3px 10px", borderRadius: 999 }}>
                    مطالعه: {toFa(hours)} س
                  </span>
                </div>

                <div style={{ display: "flex", flexDirection: "column", gap: 6 }}>
                  {rowsEx.map((row, i) => {
                    const sub = curriculumById(row.subjectId);
                    const pct = subjectPercent(row, ex);
                    const h = studyHoursForSubjectBeforeExam(row.subjectId, ex, exams, sessions);
                    return (
                      <div key={i} style={{
                        background: COLORS.bg, borderRadius: 10, padding: "8px 10px", border: `1px solid ${COLORS.border}`,
                      }}>
                        <div style={{ display: "flex", alignItems: "center", gap: 6, marginBottom: 4 }}>
                          <span style={{ width: 7, height: 7, borderRadius: 99, background: sub?.color || COLORS.muted }} />
                          <span style={{ fontSize: 12.5, fontWeight: 700 }}>{sub?.name || "—"}</span>
                          <span style={{ fontSize: 10.5, color: COLORS.muted, marginRight: "auto" }}>ضریب {toFa(row.coefficient || 1)}</span>
                        </div>
                        {row.topic ? <div style={{ fontSize: 11, color: COLORS.muted, marginBottom: 3 }}>📌 {row.topic}</div> : null}
                        <div style={{ fontSize: 11.5, color: COLORS.text }}>
                          {row.score == null || row.score === ""
                            ? (pct ? `${toFa(pct)}٪ (از تحلیل)` : "بدون نمره")
                            : row.scoreType === "percent"
                            ? `${toFa(row.score)}٪`
                            : `${toFa(row.score)} از ${toFa(row.maxScore)} (${toFa(Math.round(pct))}٪)`}
                          <span style={{ color: COLORS.muted }}> · مطالعه {toFa(h)} س</span>
                        </div>
                      </div>
                    );
                  })}
                </div>
                <ExamSubjectsChart exam={ex} allExams={exams} sessions={sessions} />
                {ex.note ? <div style={{ fontSize: 11.5, color: COLORS.muted, marginTop: 8 }}>{ex.note}</div> : null}
                <button type="button" onClick={() => setDetailId(ex.id)} className="kn-btn" style={{
                  width: "100%", marginTop: 10, background: COLORS.accent, color: "#1a1400", border: "none",
                  borderRadius: 10, padding: "10px 0", fontWeight: 800, fontSize: 13,
                }}>تحلیل آزمون</button>
              </div>
            );
          })}
        </div>
      )}
        </>
      )}
    </div>
  );
}




const PLAN_DAYS = [
  { id: 0, label: "شنبه" },
  { id: 1, label: "یکشنبه" },
  { id: 2, label: "دوشنبه" },
  { id: 3, label: "سه‌شنبه" },
  { id: 4, label: "چهارشنبه" },
  { id: 5, label: "پنجشنبه" },
  { id: 6, label: "جمعه" },
];

async function generateWeeklyPlanPdf(plan, profile) {
  const title = plan?.title || "برنامه مطالعاتی هفتگی";
  const items = plan?.items || [];
  const name = profile?.name || profile?.username || "";

  const dayBlocks = PLAN_DAYS.map((day) => {
    const list = items.filter((it) => it.dayId === day.id).sort((a, b) => (a.order || 0) - (b.order || 0));
    const studyMin = list.reduce((a, it) => a + (Number(it.durationMin) || 0), 0);
    const restMin = list.reduce((a, it) => a + (Number(it.restMin) || 0), 0);
    return { day, list, studyMin, restMin };
  });

  const totalStudy = dayBlocks.reduce((a, d) => a + d.studyMin, 0);
  const totalRest = dayBlocks.reduce((a, d) => a + d.restMin, 0);

  const wrap = document.createElement("div");
  wrap.style.cssText = "position:fixed;left:-9999px;top:0;width:900px;background:#fff;color:#111;font-family:Vazirmatn,Tahoma,sans-serif;direction:rtl;padding:28px;box-sizing:border-box;";

  let bodyHtml = "";
  dayBlocks.forEach(({ day, list, studyMin, restMin }) => {
    bodyHtml += `
      <div style="margin-bottom:18px;page-break-inside:avoid;">
        <div style="background:#14172B;color:#fff;border-radius:10px 10px 0 0;padding:10px 14px;display:flex;justify-content:space-between;align-items:center;">
          <span style="font-weight:800;font-size:14px;">${day.label}</span>
          <span style="font-size:11.5px;opacity:.9;">مطالعه ${formatDuration(studyMin)} · استراحت ${formatDuration(restMin)}</span>
        </div>
        <table style="width:100%;border-collapse:collapse;font-size:12px;border:1px solid #e5e7eb;border-top:none;">
          <thead>
            <tr style="background:#f3f4f6;">
              <th style="text-align:right;padding:8px 10px;width:40px;">#</th>
              <th style="text-align:right;padding:8px 10px;">درس‌ها</th>
              <th style="text-align:right;padding:8px 10px;width:90px;">مدت باکس</th>
              <th style="text-align:right;padding:8px 10px;width:90px;">استراحت بعد</th>
              <th style="text-align:right;padding:8px 10px;">یادداشت</th>
            </tr>
          </thead>
          <tbody>
            ${list.length ? list.map((it, idx) => {
              const names = (it.subjectIds || []).map((id) => curriculumById(id)?.name || id).join("، ") || "—";
              return `<tr style="background:${idx % 2 ? "#fafafa" : "#fff"};">
                <td style="padding:8px 10px;border-top:1px solid #eee;font-weight:700;">${toFa(idx + 1)}</td>
                <td style="padding:8px 10px;border-top:1px solid #eee;">${names}</td>
                <td style="padding:8px 10px;border-top:1px solid #eee;">${formatDuration(it.durationMin || 0)}</td>
                <td style="padding:8px 10px;border-top:1px solid #eee;">${it.restMin ? formatDuration(it.restMin) : "—"}</td>
                <td style="padding:8px 10px;border-top:1px solid #eee;color:#555;">${it.note || "—"}</td>
              </tr>`;
            }).join("") : `<tr><td colspan="5" style="padding:12px;color:#888;text-align:center;">باکسی تعریف نشده</td></tr>`}
          </tbody>
        </table>
      </div>`;
  });

  wrap.innerHTML = `
    <div style="border-bottom:3px solid #f0b429;padding-bottom:14px;margin-bottom:16px;">
      <div style="font-size:22px;font-weight:800;color:#14172B;">${title}</div>
      ${name ? `<div style="font-size:13px;color:#555;margin-top:6px;">${name}</div>` : ""}
      <div style="font-size:12.5px;color:#666;margin-top:6px;">
        مجموع مطالعه هفته: <b>${formatDuration(totalStudy)}</b>
        · مجموع استراحت: <b>${formatDuration(totalRest)}</b>
        · تعداد باکس: <b>${toFa(items.length)}</b>
      </div>
    </div>
    ${bodyHtml}
    <div style="margin-top:8px;font-size:11px;color:#888;text-align:center;">تولیدشده توسط اپ کانون</div>
  `;
  document.body.appendChild(wrap);
  try {
    if (typeof html2canvas !== "function") throw new Error("html2canvas missing");
    const canvas = await html2canvas(wrap, { scale: 2, backgroundColor: "#ffffff", useCORS: true, logging: false });
    const img = canvas.toDataURL("image/jpeg", 0.92);
    const jspdfNS = window.jspdf || window.jsPDF;
    if (!jspdfNS || !jspdfNS.jsPDF) throw new Error("jsPDF missing");
    const { jsPDF } = jspdfNS;
    const pdf = new jsPDF({ orientation: "p", unit: "mm", format: "a4" });
    const pageW = pdf.internal.pageSize.getWidth();
    const pageH = pdf.internal.pageSize.getHeight();
    const margin = 8;
    const imgW = pageW - margin * 2;
    const imgH = (canvas.height * imgW) / canvas.width;
    let heightLeft = imgH;
    let position = margin;
    pdf.addImage(img, "JPEG", margin, position, imgW, imgH);
    heightLeft -= (pageH - margin * 2);
    while (heightLeft > 0) {
      position = margin - (imgH - heightLeft);
      pdf.addPage();
      pdf.addImage(img, "JPEG", margin, position, imgW, imgH);
      heightLeft -= (pageH - margin * 2);
    }
    pdf.save("kanoon-weekly-plan.pdf");
  } finally {
    document.body.removeChild(wrap);
  }
}

function WeeklyPlanView({ weeklyPlan, setWeeklyPlan, profile }) {
  const plan = weeklyPlan || { title: "برنامه مطالعاتی هفتگی", items: [] };
  const items = plan.items || [];
  const [dayId, setDayId] = useState(0);
  const [adding, setAdding] = useState(false);
  const [editingId, setEditingId] = useState(null);
  const [subjectIds, setSubjectIds] = useState([]);
  const [durH, setDurH] = useState("1");
  const [durM, setDurM] = useState("15");
  const [restM, setRestM] = useState("15");
  const [note, setNote] = useState("");
  const [titleEdit, setTitleEdit] = useState(plan.title || "");
  const [pdfBusy, setPdfBusy] = useState(false);
  const [msg, setMsg] = useState("");

  const dayItems = items.filter((it) => it.dayId === dayId).sort((a, b) => (a.order || 0) - (b.order || 0));
  const dayStudy = dayItems.reduce((a, it) => a + (Number(it.durationMin) || 0), 0);
  const dayRest = dayItems.reduce((a, it) => a + (Number(it.restMin) || 0), 0);

  function resetForm() {
    setSubjectIds([]);
    setDurH("1");
    setDurM("15");
    setRestM("15");
    setNote("");
    setEditingId(null);
    setAdding(false);
  }

  function openEdit(it) {
    setEditingId(it.id);
    setAdding(true);
    setSubjectIds(it.subjectIds || []);
    const total = Number(it.durationMin) || 0;
    setDurH(String(Math.floor(total / 60)));
    setDurM(String(total % 60));
    setRestM(String(it.restMin != null ? it.restMin : 15));
    setNote(it.note || "");
  }

  function saveItem() {
    if (subjectIds.length === 0) {
      setMsg("حداقل یک درس انتخاب کن");
      return;
    }
    const h = parseInt(durH || "0", 10) || 0;
    const m = parseInt(durM || "0", 10) || 0;
    const durationMin = h * 60 + m;
    if (durationMin < 1) {
      setMsg("مدت باکس باید بیشتر از صفر باشد");
      return;
    }
    const restMin = Math.max(0, parseInt(restM || "0", 10) || 0);
    setWeeklyPlan((prev) => {
      const base = prev || { title: "برنامه مطالعاتی هفتگی", items: [] };
      const list = [...(base.items || [])];
      if (editingId) {
        const idx = list.findIndex((x) => x.id === editingId);
        if (idx >= 0) {
          list[idx] = { ...list[idx], subjectIds: [...subjectIds], durationMin, restMin, note: note.trim(), dayId };
        }
      } else {
        const order = list.filter((x) => x.dayId === dayId).length;
        list.push({
          id: Date.now().toString(36) + Math.random().toString(36).slice(2, 6),
          dayId,
          subjectIds: [...subjectIds],
          durationMin,
          restMin,
          note: note.trim(),
          order,
        });
      }
      return { ...base, items: list };
    });
    setMsg(editingId ? "باکس ویرایش شد" : "باکس اضافه شد");
    resetForm();
  }

  function deleteItem(id) {
    if (!confirm("این باکس از برنامه حذف شود؟")) return;
    setWeeklyPlan((prev) => {
      const base = prev || { title: "برنامه مطالعاتی هفتگی", items: [] };
      return { ...base, items: (base.items || []).filter((x) => x.id !== id) };
    });
  }

  function saveTitle() {
    setWeeklyPlan((prev) => ({ ...(prev || { items: [] }), title: titleEdit.trim() || "برنامه مطالعاتی هفتگی" }));
    setMsg("عنوان ذخیره شد");
  }

  async function downloadPdf() {
    setPdfBusy(true);
    setMsg("");
    try {
      await generateWeeklyPlanPdf(plan, profile);
      setMsg("PDF برنامه دانلود شد");
    } catch (e) {
      console.error(e);
      setMsg("ساخت PDF ناموفق بود");
    }
    setPdfBusy(false);
  }

  return (
    <div className="kn-fade">
      <div style={{ marginBottom: 14 }}>
        <div style={{ fontSize: 15, fontWeight: 800, marginBottom: 4 }}>برنامه مطالعاتی هفتگی</div>
        <div style={{ fontSize: 12, color: COLORS.muted, lineHeight: 1.7 }}>
          برای هر روز باکس تعریف کن (درس + مدت + استراحت بین باکس‌ها) و هر وقت خواستی به آن مراجعه کن یا PDF بگیر.
        </div>
      </div>

      <div style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14, padding: 12, marginBottom: 14 }}>
        <div style={{ fontSize: 11, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>عنوان برنامه</div>
        <div style={{ display: "flex", gap: 8 }}>
          <input value={titleEdit} onChange={(e) => setTitleEdit(e.target.value)}
            style={{ flex: 1, background: COLORS.surface2, border: `1px solid ${COLORS.border}`, borderRadius: 10, padding: "8px 12px", color: COLORS.text, fontSize: 13, outline: "none" }} />
          <button onClick={saveTitle} className="kn-btn" style={{ background: COLORS.accent, color: "#1a1400", border: "none", borderRadius: 10, padding: "0 14px", fontWeight: 700, fontSize: 12 }}>ذخیره</button>
        </div>
      </div>

      <div style={{ display: "flex", gap: 6, overflowX: "auto", marginBottom: 12, paddingBottom: 2 }}>
        {PLAN_DAYS.map((d) => {
          const count = items.filter((it) => it.dayId === d.id).length;
          const on = dayId === d.id;
          return (
            <button key={d.id} onClick={() => { setDayId(d.id); resetForm(); }} className="kn-btn" style={{
              flex: "0 0 auto", border: `1.5px solid ${on ? COLORS.accent : COLORS.border}`,
              background: on ? COLORS.accent + "22" : COLORS.surface,
              color: on ? COLORS.accentSoft : COLORS.muted,
              borderRadius: 10, padding: "8px 12px", fontSize: 12, fontWeight: 700,
            }}>
              {d.label}{count ? ` (${toFa(count)})` : ""}
            </button>
          );
        })}
      </div>

      <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 10 }}>
        <div style={{ fontSize: 12, color: COLORS.muted }}>
          امروزِ برنامه: مطالعه {formatDuration(dayStudy)} · استراحت {formatDuration(dayRest)}
        </div>
        <button onClick={() => { if (adding && !editingId) resetForm(); else { setAdding(true); setEditingId(null); } }} className="kn-btn" style={{
          background: COLORS.accent, color: "#1a1400", border: "none", borderRadius: 10, padding: "7px 12px", fontSize: 12, fontWeight: 700,
          display: "flex", alignItems: "center", gap: 4,
        }}>
          <Plus size={14} /> باکس جدید
        </button>
      </div>

      {adding && (
        <div style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14, padding: 14, marginBottom: 14 }}>
          <div style={{ fontSize: 13, fontWeight: 700, marginBottom: 10 }}>{editingId ? "ویرایش باکس" : "افزودن باکس"} — {PLAN_DAYS.find((d) => d.id === dayId)?.label}</div>
          <div style={{ marginBottom: 10 }}>
            <div style={{ fontSize: 11, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>درس‌ها</div>
            <GroupedSubjectPicker items={CURRICULUM} mode="multi" value={subjectIds} onChange={setSubjectIds} />
          </div>
          <div style={{ fontSize: 11, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>مدت مطالعه باکس</div>
          <DurationWheel hours={durH} minutes={durM} onHours={setDurH} onMinutes={setDurM} />
          <div style={{ fontSize: 11, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>استراحت بعد از باکس (تا ۵ ساعت، گام ۵ دقیقه)</div>
          <MinutesWheel minutes={restM} onChange={setRestM} label="استراحت" maxMin={300} />
          <input value={note} onChange={(e) => setNote(e.target.value)} placeholder="یادداشت (اختیاری)"
            style={{ width: "100%", background: COLORS.surface2, border: `1px solid ${COLORS.border}`, borderRadius: 10, padding: "8px 12px", color: COLORS.text, fontSize: 12.5, outline: "none", marginBottom: 10 }} />
          <div style={{ display: "flex", gap: 8 }}>
            <button onClick={saveItem} className="kn-btn" style={{ flex: 1, background: COLORS.accent, color: "#1a1400", border: "none", borderRadius: 10, padding: "10px 0", fontWeight: 800, fontSize: 13 }}>
              {editingId ? "ذخیره تغییرات" : "افزودن به برنامه"}
            </button>
            <button onClick={resetForm} className="kn-btn" style={{ flex: 1, background: "none", border: `1px solid ${COLORS.border}`, borderRadius: 10, padding: "10px 0", color: COLORS.muted, fontSize: 13 }}>انصراف</button>
          </div>
        </div>
      )}

      {dayItems.length === 0 ? (
        <div style={{ textAlign: "center", color: COLORS.muted, padding: 28, background: COLORS.surface, borderRadius: 14, border: `1px solid ${COLORS.border}`, marginBottom: 14 }}>
          برای این روز هنوز باکسی نذاشتی.
        </div>
      ) : (
        <div style={{ display: "flex", flexDirection: "column", gap: 8, marginBottom: 14 }}>
          {dayItems.map((it, idx) => {
            const names = (it.subjectIds || []).map((id) => curriculumById(id)?.name || id).join("، ");
            return (
              <div key={it.id} style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 12, padding: "12px 14px" }}>
                <div style={{ display: "flex", justifyContent: "space-between", gap: 8, marginBottom: 6 }}>
                  <div style={{ fontSize: 13, fontWeight: 700 }}>باکس {toFa(idx + 1)}</div>
                  <div style={{ display: "flex", gap: 6 }}>
                    <button onClick={() => openEdit(it)} className="kn-btn" style={{ background: "none", border: "none", color: COLORS.accent, fontSize: 11.5, fontWeight: 700 }}>ویرایش</button>
                    <button onClick={() => deleteItem(it.id)} className="kn-btn" style={{ background: "none", border: "none", color: COLORS.danger, fontSize: 11.5, fontWeight: 700 }}>حذف</button>
                  </div>
                </div>
                <div style={{ fontSize: 12.5, color: COLORS.text, marginBottom: 6 }}>{names || "—"}</div>
                <div style={{ display: "flex", flexWrap: "wrap", gap: 8, fontSize: 11.5, color: COLORS.muted }}>
                  <span style={{ background: COLORS.accent + "22", color: COLORS.accentSoft, padding: "2px 8px", borderRadius: 999, fontWeight: 700 }}>مطالعه {formatDuration(it.durationMin)}</span>
                  <span style={{ background: COLORS.surface2, padding: "2px 8px", borderRadius: 999 }}>استراحت بعد {it.restMin ? formatDuration(it.restMin) : "—"}</span>
                  {it.note ? <span>📝 {it.note}</span> : null}
                </div>
              </div>
            );
          })}
        </div>
      )}

      {/* خلاصه هفته */}
      <div style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14, padding: 14, marginBottom: 14 }}>
        <div style={{ fontSize: 13, fontWeight: 700, marginBottom: 10, color: COLORS.muted }}>خلاصه هفته</div>
        <div style={{ display: "flex", flexDirection: "column", gap: 6 }}>
          {PLAN_DAYS.map((d) => {
            const list = items.filter((it) => it.dayId === d.id);
            const sm = list.reduce((a, it) => a + (Number(it.durationMin) || 0), 0);
            return (
              <div key={d.id} style={{ display: "flex", justifyContent: "space-between", fontSize: 12.5, padding: "6px 0", borderBottom: `1px solid ${COLORS.border}` }}>
                <span>{d.label} · {toFa(list.length)} باکس</span>
                <span style={{ color: COLORS.muted }}>{sm ? formatDuration(sm) : "—"}</span>
              </div>
            );
          })}
        </div>
      </div>

      <button onClick={downloadPdf} disabled={pdfBusy || items.length === 0} className="kn-btn" style={{
        width: "100%", background: items.length ? COLORS.accent : COLORS.surface2,
        color: items.length ? "#1a1400" : COLORS.muted, border: "none",
        borderRadius: 12, padding: "12px 0", fontSize: 14, fontWeight: 800, marginBottom: 8,
      }}>
        {pdfBusy ? "در حال ساخت PDF..." : "دانلود جدول برنامه (PDF)"}
      </button>
      {msg && <div style={{ fontSize: 12, color: COLORS.accentSoft, textAlign: "center" }}>{msg}</div>}
    </div>
  );
}


const PERIODS = [
  { id: "daily", label: "روزانه" },
  { id: "weekly", label: "هفتگی" },
  { id: "monthly", label: "ماهانه" },
  { id: "custom", label: "بازه دلخواه" },
];

function keysBetween(fromKey, toKey) {
  const out = [];
  if (!fromKey || !toKey) return out;
  let a = fromKey;
  let b = toKey;
  if (a > b) { const t = a; a = b; b = t; }
  const d = new Date(a + "T00:00:00");
  const end = new Date(b + "T00:00:00");
  while (d <= end) {
    out.push(todayKey(d));
    d.setDate(d.getDate() + 1);
  }
  return out;
}

function CompetitionView({ profile }) {
  const [period, setPeriod] = useState("weekly");
  const [rows, setRows] = useState(null);
  const [refreshing, setRefreshing] = useState(false);
  const [errored, setErrored] = useState(false);
  const [customFrom, setCustomFrom] = useState(() => {
    const d = new Date(); d.setDate(d.getDate() - 6); return todayKey(d);
  });
  const [customTo, setCustomTo] = useState(() => todayKey());

  async function fetchLeaderboard() {
    setRefreshing(true);
    setErrored(false);
    try {
      const entries = await fetchLeaderboardFromFirebase();
      setRows(entries);
    } catch (e) {
      console.error(e);
      setErrored(true);
      setRows((prev) => prev ?? []);
    }
    setRefreshing(false);
  }

  useEffect(() => { fetchLeaderboard(); }, []);

  const periodKeys = useMemo(() => {
    if (period === "daily") return [todayKey()];
    if (period === "weekly") return getCurrentWeekDates().map((d) => todayKey(d));
    if (period === "monthly") return currentJalaliMonthKeys();
    return keysBetween(customFrom, customTo);
  }, [period, customFrom, customTo]);

  const ranked = useMemo(() => {
    if (!rows) return [];
    const keySet = periodKeys;
    const computed = rows.map((r) => {
      let minutes = 0, tests = 0, exercises = 0;
      keySet.forEach((k) => {
        const d = r.dailyStats?.[k];
        if (d) { minutes += d.minutes || 0; tests += d.tests || 0; exercises += d.exercises || 0; }
      });
      return { userId: r.userId, name: r.name, grade: r.grade, minutes, tests, exercises };
    });
    return computed.sort((a, b) => b.minutes - a.minutes || b.tests - a.tests);
  }, [rows, periodKeys]);

  const CHART_COLORS = ["#f0b429", "#2dd4a8", "#60a5fa", "#f472b6", "#a78bfa", "#fb923c", "#34d399", "#f87171"];

  const [selectedChartUsers, setSelectedChartUsers] = useState([]);

  // پیش‌فرض: خودت + بقیه تا ۳ نفر اول رتبه‌بندی
  useEffect(() => {
    if (!rows || rows.length === 0) return;
    if (selectedChartUsers.length > 0) return;
    const ids = [];
    if (profile?.userId && rows.find((r) => r.userId === profile.userId)) {
      ids.push(profile.userId);
    }
    ranked.forEach((r) => {
      if (ids.length >= 3) return;
      if (!ids.includes(r.userId)) ids.push(r.userId);
    });
    if (ids.length === 0 && rows[0]) ids.push(rows[0].userId);
    setSelectedChartUsers(ids);
  }, [rows, ranked, profile]);

  function toggleChartUser(userId) {
    setSelectedChartUsers((prev) => {
      if (prev.includes(userId)) {
        if (prev.length <= 1) return prev; // حداقل یک نفر
        return prev.filter((id) => id !== userId);
      }
      return [...prev, userId];
    });
  }

  const selectedPeople = useMemo(() => {
    if (!rows) return [];
    return selectedChartUsers
      .map((id, idx) => {
        const r = rows.find((x) => x.userId === id);
        if (!r) return null;
        return { ...r, color: CHART_COLORS[idx % CHART_COLORS.length] };
      })
      .filter(Boolean);
  }, [rows, selectedChartUsers]);

  // محور X: روزها یا هفته‌های ماه — برای هر نقطه، مقدار هر فرد جدا
  const chartBuckets = useMemo(() => {
    if (!rows || periodKeys.length === 0) return [];
    if (period === "monthly") {
      const weeks = {};
      periodKeys.forEach((k) => {
        const d = new Date(k + "T00:00:00");
        const { jd } = gregorianToJalaliYMD(d.getFullYear(), d.getMonth() + 1, d.getDate());
        const weekIdx = Math.ceil(jd / 7);
        if (!weeks[weekIdx]) weeks[weekIdx] = { key: "w" + weekIdx, label: "هفته " + toFa(weekIdx), order: weekIdx, keys: [] };
        weeks[weekIdx].keys.push(k);
      });
      return Object.values(weeks).sort((a, b) => a.order - b.order);
    }
    return periodKeys.map((k) => {
      const d = new Date(k + "T00:00:00");
      const label = period === "weekly" ? WEEKDAY_FA[d.getDay()] : toJalaliShort(d);
      return { key: k, label, keys: [k] };
    });
  }, [rows, periodKeys, period]);

  const chartSeries = useMemo(() => {
    return chartBuckets.map((bucket) => {
      const values = {};
      selectedPeople.forEach((p) => {
        let minutes = 0;
        bucket.keys.forEach((k) => {
          const st = p.dailyStats?.[k];
          if (st) minutes += st.minutes || 0;
        });
        values[p.userId] = minutes;
      });
      return { key: bucket.key, label: bucket.label, values };
    });
  }, [chartBuckets, selectedPeople]);

  const maxChart = Math.max(
    1,
    ...chartSeries.flatMap((b) => Object.values(b.values).map((v) => Number(v) || 0))
  );

  const periodLabel =
    period === "daily" ? "امروز" :
    period === "weekly" ? "این هفته (شنبه تا جمعه)" :
    period === "monthly" ? "این ماه" :
    "بازه انتخابی";

  const medalColor = (rank) => (rank === 0 ? COLORS.accent : rank === 1 ? COLORS.silver : rank === 2 ? COLORS.bronze : COLORS.muted);
  const showChart = period === "weekly" || period === "monthly" || period === "custom";

  return (
    <div className="kn-fade">
      <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 14 }}>
        <div>
          <div style={{ fontSize: 15, fontWeight: 700, display: "flex", alignItems: "center", gap: 6 }}>
            <Trophy size={17} color={COLORS.accent} /> رقابت
          </div>
          <div style={{ fontSize: 11.5, color: COLORS.muted, marginTop: 2 }}>{periodLabel}</div>
        </div>
        <button onClick={fetchLeaderboard} disabled={refreshing} className="kn-btn" style={{
          background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 10,
          padding: 8, color: COLORS.text, display: "flex",
        }}>
          <RefreshCw size={15} className={refreshing ? "kn-spin" : ""} />
        </button>
      </div>

      <div style={{ display: "flex", gap: 6, marginBottom: 12, flexWrap: "wrap" }}>
        {PERIODS.map((p) => (
          <button key={p.id} onClick={() => setPeriod(p.id)} className="kn-btn" style={{
            flex: 1, minWidth: "22%", border: `1.5px solid ${period === p.id ? COLORS.accent : COLORS.border}`,
            background: period === p.id ? COLORS.accent + "22" : "transparent",
            color: period === p.id ? COLORS.accentSoft : COLORS.muted,
            borderRadius: 10, padding: "8px 0", fontSize: 12, fontWeight: 700,
          }}>{p.label}</button>
        ))}
      </div>

      {period === "custom" && (
        <div style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 12, padding: 12, marginBottom: 14 }}>
          <div style={{ fontSize: 11.5, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>از تاریخ</div>
          <div style={{ marginBottom: 10 }}>
            <JalaliDatePicker value={customFrom} onChange={setCustomFrom} />
          </div>
          <div style={{ fontSize: 11.5, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>تا تاریخ</div>
          <JalaliDatePicker value={customTo} onChange={setCustomTo} />
        </div>
      )}

      {errored && (
        <div style={{ fontSize: 11.5, color: COLORS.danger, background: COLORS.danger + "18", borderRadius: 10, padding: "8px 12px", marginBottom: 12 }}>
          دریافت اطلاعات رقابت با خطا مواجه شد. دوباره تلاش کن.
        </div>
      )}

      {rows === null ? (
        <div style={{ textAlign: "center", color: COLORS.muted, padding: 50, fontSize: 13 }}>در حال بارگذاری رتبه‌بندی...</div>
      ) : (
        <>
          {showChart && (
            <div style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14, padding: 14, marginBottom: 14 }}>
              <div style={{ fontSize: 13, fontWeight: 700, marginBottom: 8, color: COLORS.muted }}>
                {period === "monthly" ? "مقایسه افراد · هفته‌های ماه" : "مقایسه افراد · روزها"}
              </div>
              <div style={{ fontSize: 11.5, color: COLORS.muted, marginBottom: 8 }}>افراد را برای نمودار انتخاب کن (چند نفر همزمان):</div>
              <div style={{ display: "flex", flexWrap: "wrap", gap: 6, marginBottom: 12 }}>
                {(rows || []).map((r) => {
                  const on = selectedChartUsers.includes(r.userId);
                  const colorIdx = selectedChartUsers.indexOf(r.userId);
                  const col = on ? CHART_COLORS[colorIdx % CHART_COLORS.length] : COLORS.muted;
                  return (
                    <button key={r.userId} type="button" onClick={() => toggleChartUser(r.userId)} className="kn-btn" style={{
                      border: `1.5px solid ${on ? col : COLORS.border}`,
                      background: on ? col + "22" : COLORS.bg,
                      color: on ? col : COLORS.muted,
                      borderRadius: 999, padding: "5px 10px", fontSize: 11.5, fontWeight: 700,
                    }}>
                      {r.name}{profile && r.userId === profile.userId ? " (تو)" : ""}
                    </button>
                  );
                })}
              </div>

              {selectedPeople.length === 0 || chartSeries.length === 0 ? (
                <div style={{ fontSize: 12, color: COLORS.muted, textAlign: "center", padding: 20 }}>کسی انتخاب نشده</div>
              ) : (
                <>
                  <div style={{ display: "flex", alignItems: "flex-end", gap: 6, height: 160, overflowX: "auto", paddingBottom: 4 }}>
                    {chartSeries.map((bucket) => (
                      <div key={bucket.key} style={{ flex: "1 0 auto", minWidth: Math.max(36, selectedPeople.length * 14), display: "flex", flexDirection: "column", alignItems: "center", height: "100%", justifyContent: "flex-end" }}>
                        <div style={{ display: "flex", alignItems: "flex-end", gap: 2, height: "100%", width: "100%", justifyContent: "center" }}>
                          {selectedPeople.map((p) => {
                            const mins = bucket.values[p.userId] || 0;
                            const h = Math.round((mins / maxChart) * 100);
                            return (
                              <div key={p.userId} title={`${p.name}: ${formatDuration(mins)}`} style={{
                                width: Math.max(6, Math.min(16, 40 / selectedPeople.length)),
                                height: `${Math.max(h, mins > 0 ? 3 : 0)}%`,
                                background: p.color,
                                borderRadius: "3px 3px 0 0",
                              }} />
                            );
                          })}
                        </div>
                        <div style={{ fontSize: 9.5, color: COLORS.muted, marginTop: 4, textAlign: "center", whiteSpace: "nowrap" }}>{bucket.label}</div>
                      </div>
                    ))}
                  </div>
                  <div style={{ display: "flex", flexWrap: "wrap", gap: 10, marginTop: 10 }}>
                    {selectedPeople.map((p) => (
                      <span key={p.userId} style={{ display: "inline-flex", alignItems: "center", gap: 5, fontSize: 11, color: COLORS.muted }}>
                        <span style={{ width: 9, height: 9, borderRadius: 2, background: p.color }} />
                        {p.name}
                      </span>
                    ))}
                  </div>
                </>
              )}
            </div>
          )}

          {ranked.length === 0 ? (
            <div style={{ textAlign: "center", color: COLORS.muted, padding: "40px 20px" }}>
              <div style={{ marginBottom: 10, display: "flex", justifyContent: "center" }}>
                <Users size={30} color={COLORS.muted} />
              </div>
              <div style={{ fontSize: 13 }}>برای این بازه هنوز مطالعه‌ای ثبت نشده.</div>
            </div>
          ) : (
            <div style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14, overflow: "hidden" }}>
              <div style={{
                display: "grid", gridTemplateColumns: "34px 1fr 78px 46px 46px", gap: 4,
                padding: "9px 12px", borderBottom: `1px solid ${COLORS.border}`,
                fontSize: 10.5, color: COLORS.muted, fontWeight: 700,
              }}>
                <span>رتبه</span>
                <span>نام</span>
                <span style={{ textAlign: "left" }}>مطالعه</span>
                <span style={{ textAlign: "left" }}>تست</span>
                <span style={{ textAlign: "left" }}>تمرین</span>
              </div>
              {ranked.map((r, i) => {
                const isMe = profile && r.userId === profile.userId;
                return (
                  <div key={r.userId} style={{
                    display: "grid", gridTemplateColumns: "34px 1fr 78px 46px 46px", gap: 4,
                    alignItems: "center", padding: "10px 12px",
                    background: isMe ? COLORS.accent + "14" : "transparent",
                    borderBottom: `1px solid ${COLORS.border}`,
                  }}>
                    <span style={{ display: "flex", alignItems: "center", gap: 2 }}>
                      {i < 3 ? <Medal size={15} color={medalColor(i)} /> : null}
                      <span style={{ fontSize: 12, fontWeight: 700, color: i < 3 ? medalColor(i) : COLORS.muted }}>{toFa(i + 1)}</span>
                    </span>
                    <span style={{ minWidth: 0, overflow: "hidden" }}>
                      <div style={{ fontSize: 12.5, fontWeight: isMe ? 800 : 600, color: isMe ? COLORS.accentSoft : COLORS.text, overflow: "hidden", textOverflow: "ellipsis", whiteSpace: "nowrap" }}>
                        {r.name}{isMe ? " (خودت)" : ""}
                      </div>
                      <div style={{ fontSize: 10, color: COLORS.muted }}>{r.grade}</div>
                    </span>
                    <span style={{ fontSize: 11.5, fontWeight: 700, textAlign: "left" }}>{formatDuration(r.minutes)}</span>
                    <span style={{ fontSize: 11.5, color: COLORS.muted, textAlign: "left" }}>{toFa(r.tests)}</span>
                    <span style={{ fontSize: 11.5, color: COLORS.muted, textAlign: "left" }}>{toFa(r.exercises)}</span>
                  </div>
                );
              })}
            </div>
          )}
        </>
      )}
    </div>
  );
}

// ================= ANALYTICS =================


async function renderHtmlToCanvas(html, widthPx = 800) {
  const wrap = document.createElement("div");
  wrap.style.cssText = `position:fixed;left:-9999px;top:0;width:${widthPx}px;background:#ffffff;color:#111;font-family:Vazirmatn,Tahoma,sans-serif;direction:rtl;padding:24px;box-sizing:border-box;`;
  wrap.innerHTML = html;
  document.body.appendChild(wrap);
  try {
    if (typeof html2canvas !== "function") throw new Error("html2canvas در دسترس نیست");
    return await html2canvas(wrap, { scale: 2, backgroundColor: "#ffffff", useCORS: true, logging: false });
  } finally {
    document.body.removeChild(wrap);
  }
}

async function appendCanvasPage(pdf, canvas, state) {
  const { pageW, pageH, margin } = state;
  const imgW = pageW - margin * 2;
  const imgH = (canvas.height * imgW) / canvas.width;
  const usable = pageH - margin * 2;
  // اگر بلوک از یک صفحه بزرگ‌تر است، به تکه‌های کامل صفحه برش بزن
  if (imgH <= usable) {
    if (state.y + imgH > pageH - margin) {
      pdf.addPage();
      state.y = margin;
    }
    const data = canvas.toDataURL("image/jpeg", 0.92);
    pdf.addImage(data, "JPEG", margin, state.y, imgW, imgH);
    state.y += imgH + 3;
    return;
  }
  // برش ارتفاعی بدون افتادن وسط صفحه قبلی
  const slicePx = Math.floor((usable * canvas.width) / imgW);
  let sy = 0;
  while (sy < canvas.height) {
    const sh = Math.min(slicePx, canvas.height - sy);
    const piece = document.createElement("canvas");
    piece.width = canvas.width;
    piece.height = sh;
    piece.getContext("2d").drawImage(canvas, 0, sy, canvas.width, sh, 0, 0, canvas.width, sh);
    if (state.y > margin + 1) {
      pdf.addPage();
      state.y = margin;
    }
    const hMm = (sh * imgW) / canvas.width;
    pdf.addImage(piece.toDataURL("image/jpeg", 0.92), "JPEG", margin, margin, imgW, hMm);
    state.y = margin + hMm + 3;
    if (sy + sh < canvas.height) {
      pdf.addPage();
      state.y = margin;
    }
    sy += sh;
  }
}

async function generateStudyReportPdf({ profile, sessions, fromKey, toKey }) {
  const keys = keysBetween(fromKey, toKey);
  const keySet = new Set(keys);
  const filtered = (sessions || []).filter((s) => keySet.has(s.date));
  const totalMin = filtered.reduce((a, s) => a + (Number(s.minutes) || 0), 0);
  const totalTests = filtered.reduce((a, s) => a + (s.exerciseType === "تست" ? (Number(s.exerciseCount) || 0) : 0), 0);
  const totalEx = filtered.reduce((a, s) => a + (s.exerciseType === "تمرین" ? (Number(s.exerciseCount) || 0) : 0), 0);

  const bySubject = CURRICULUM.map((c) => {
    const mins = filtered.filter((s) => s.subjectId === c.id).reduce((a, s) => a + (Number(s.minutes) || 0), 0);
    const tests = filtered.filter((s) => s.subjectId === c.id && s.exerciseType === "تست").reduce((a, s) => a + (Number(s.exerciseCount) || 0), 0);
    const exercises = filtered.filter((s) => s.subjectId === c.id && s.exerciseType === "تمرین").reduce((a, s) => a + (Number(s.exerciseCount) || 0), 0);
    return { name: c.name, color: c.color, mins, tests, exercises };
  }).filter((x) => x.mins > 0 || x.tests > 0 || x.exercises > 0)
    .sort((a, b) => b.mins - a.mins);

  const byDay = keys.map((k) => {
    const mins = filtered.filter((s) => s.date === k).reduce((a, s) => a + (Number(s.minutes) || 0), 0);
    const d = new Date(k + "T00:00:00");
    return { key: k, label: toJalaliShort(d) + " · " + WEEKDAY_FULL_FA[d.getDay()], mins };
  });

  const sessionsSorted = filtered.slice().sort((a, b) => (a.date + String(a.timestamp)).localeCompare(b.date + String(b.timestamp)));

  const name = profile?.name || profile?.username || "کاربر";
  const grade = profile?.grade || "—";
  const fromLabel = toJalaliShort(new Date(fromKey + "T00:00:00"));
  const toLabel = toJalaliShort(new Date(toKey + "T00:00:00"));

  const jspdfNS = window.jspdf || window.jsPDF;
  if (!jspdfNS || !jspdfNS.jsPDF) throw new Error("jsPDF در دسترس نیست");
  const { jsPDF } = jspdfNS;
  const pdf = new jsPDF({ orientation: "p", unit: "mm", format: "a4" });
  const state = {
    pageW: pdf.internal.pageSize.getWidth(),
    pageH: pdf.internal.pageSize.getHeight(),
    margin: 10,
    y: 10,
  };

  // بلوک ۱: هدر + خلاصه + تفکیک درس
  const headerHtml = `
    <div style="border-bottom:3px solid #f0b429;padding-bottom:14px;margin-bottom:16px;">
      <div style="font-size:22px;font-weight:800;color:#14172B;">گزارش مطالعه — کانون</div>
      <div style="font-size:13px;color:#555;margin-top:6px;">${name} · پایه ${grade}</div>
      <div style="font-size:13px;color:#555;margin-top:4px;">بازه: ${fromLabel} تا ${toLabel}</div>
    </div>
    <div style="display:flex;gap:12px;margin-bottom:16px;flex-wrap:wrap;">
      <div style="flex:1;min-width:140px;background:#f7f7fb;border-radius:12px;padding:12px 14px;">
        <div style="font-size:11px;color:#666;">مجموع مطالعه</div>
        <div style="font-size:18px;font-weight:800;margin-top:4px;">${formatDuration(totalMin)}</div>
      </div>
      <div style="flex:1;min-width:140px;background:#f7f7fb;border-radius:12px;padding:12px 14px;">
        <div style="font-size:11px;color:#666;">تعداد جلسات</div>
        <div style="font-size:18px;font-weight:800;margin-top:4px;">${toFa(filtered.length)}</div>
      </div>
      <div style="flex:1;min-width:140px;background:#f7f7fb;border-radius:12px;padding:12px 14px;">
        <div style="font-size:11px;color:#666;">تست / تمرین</div>
        <div style="font-size:18px;font-weight:800;margin-top:4px;">${toFa(totalTests)} / ${toFa(totalEx)}</div>
      </div>
    </div>
    <div style="font-size:15px;font-weight:800;margin-bottom:10px;color:#14172B;">به تفکیک درس</div>
    <table style="width:100%;border-collapse:collapse;font-size:12.5px;">
      <thead><tr style="background:#14172B;color:#fff;">
        <th style="text-align:right;padding:8px 10px;">درس</th>
        <th style="text-align:right;padding:8px 10px;">مدت</th>
        <th style="text-align:right;padding:8px 10px;">تست</th>
        <th style="text-align:right;padding:8px 10px;">تمرین</th>
      </tr></thead>
      <tbody>
        ${bySubject.length ? bySubject.map((s, i) => `
          <tr style="background:${i % 2 ? "#f7f7fb" : "#fff"};">
            <td style="padding:8px 10px;border-bottom:1px solid #eee;">${s.name}</td>
            <td style="padding:8px 10px;border-bottom:1px solid #eee;">${formatDuration(s.mins)}</td>
            <td style="padding:8px 10px;border-bottom:1px solid #eee;">${toFa(s.tests)}</td>
            <td style="padding:8px 10px;border-bottom:1px solid #eee;">${toFa(s.exercises)}</td>
          </tr>`).join("") : `<tr><td colspan="4" style="padding:12px;color:#888;">در این بازه درسی ثبت نشده</td></tr>`}
      </tbody>
    </table>`;
  await appendCanvasPage(pdf, await renderHtmlToCanvas(headerHtml), state);

  // بلوک روزها — هر ۲۰ ردیف یک بخش جدا
  const dayChunk = 18;
  for (let i = 0; i < byDay.length; i += dayChunk) {
    const chunk = byDay.slice(i, i + dayChunk);
    const html = `
      <div style="font-size:15px;font-weight:800;margin-bottom:10px;color:#14172B;">به تفکیک روز${i ? " (ادامه)" : ""}</div>
      <table style="width:100%;border-collapse:collapse;font-size:12.5px;">
        <thead><tr style="background:#14172B;color:#fff;">
          <th style="text-align:right;padding:8px 10px;">روز</th>
          <th style="text-align:right;padding:8px 10px;">مدت مطالعه</th>
        </tr></thead>
        <tbody>
          ${chunk.map((d, j) => `
            <tr style="background:${j % 2 ? "#f7f7fb" : "#fff"};">
              <td style="padding:8px 10px;border-bottom:1px solid #eee;">${d.label}</td>
              <td style="padding:8px 10px;border-bottom:1px solid #eee;">${d.mins ? formatDuration(d.mins) : "—"}</td>
            </tr>`).join("")}
        </tbody>
      </table>`;
    await appendCanvasPage(pdf, await renderHtmlToCanvas(html), state);
  }

  // جلسات — هر ۱۲ ردیف یک صفحه منطقی
  const sessChunk = 12;
  if (sessionsSorted.length === 0) {
    const html = `<div style="font-size:15px;font-weight:800;margin-bottom:10px;color:#14172B;">جزئیات جلسات</div>
      <div style="color:#888;padding:12px;">جلسه‌ای در این بازه نیست</div>`;
    await appendCanvasPage(pdf, await renderHtmlToCanvas(html), state);
  } else {
    for (let i = 0; i < sessionsSorted.length; i += sessChunk) {
      const chunk = sessionsSorted.slice(i, i + sessChunk);
      const html = `
        <div style="font-size:15px;font-weight:800;margin-bottom:10px;color:#14172B;">جزئیات جلسات${i ? " (ادامه)" : ""}</div>
        <table style="width:100%;border-collapse:collapse;font-size:11.5px;">
          <thead><tr style="background:#14172B;color:#fff;">
            <th style="text-align:right;padding:7px 8px;">تاریخ</th>
            <th style="text-align:right;padding:7px 8px;">درس</th>
            <th style="text-align:right;padding:7px 8px;">مبحث</th>
            <th style="text-align:right;padding:7px 8px;">مدت</th>
            <th style="text-align:right;padding:7px 8px;">تست/تمرین</th>
          </tr></thead>
          <tbody>
            ${chunk.map((s, j) => {
              const sub = curriculumById(s.subjectId);
              const ex = s.exerciseCount ? `${toFa(s.exerciseCount)} ${s.exerciseType || ""}` : "—";
              return `<tr style="background:${j % 2 ? "#f7f7fb" : "#fff"};">
                <td style="padding:7px 8px;border-bottom:1px solid #eee;">${toJalaliShort(new Date(s.date + "T00:00:00"))}</td>
                <td style="padding:7px 8px;border-bottom:1px solid #eee;">${sub?.name || "—"}</td>
                <td style="padding:7px 8px;border-bottom:1px solid #eee;">${s.topic || "—"}</td>
                <td style="padding:7px 8px;border-bottom:1px solid #eee;">${formatDuration(s.minutes)}</td>
                <td style="padding:7px 8px;border-bottom:1px solid #eee;">${ex}</td>
              </tr>`;
            }).join("")}
          </tbody>
        </table>`;
      await appendCanvasPage(pdf, await renderHtmlToCanvas(html), state);
    }
  }

  const foot = `<div style="font-size:11px;color:#888;text-align:center;padding-top:8px;">تولیدشده توسط اپ کانون</div>`;
  await appendCanvasPage(pdf, await renderHtmlToCanvas(foot), state);

  pdf.save(`kanoon-report-${fromKey}_${toKey}.pdf`);
}

function AnalyticsView({ sessions = [], boxes = [], streak = 0, settings = {}, dayOverrides = {}, profile = null }) {
  const [detailDate, setDetailDate] = useState(todayKey());
  const [reportFrom, setReportFrom] = useState(() => {
    const d = new Date(); d.setDate(d.getDate() - 6); return todayKey(d);
  });
  const [reportTo, setReportTo] = useState(() => todayKey());
  const [pdfBusy, setPdfBusy] = useState(false);
  const [pdfMsg, setPdfMsg] = useState("");

  async function handleDownloadPdf() {
    setPdfBusy(true);
    setPdfMsg("");
    try {
      let from = reportFrom;
      let to = reportTo;
      if (from > to) { const t = from; from = to; to = t; }
      await generateStudyReportPdf({ profile, sessions, fromKey: from, toKey: to });
      setPdfMsg("گزارش PDF دانلود شد");
    } catch (e) {
      console.error(e);
      setPdfMsg("ساخت PDF ناموفق بود. اتصال اینترنت و کتابخانه‌ها را چک کن.");
    }
    setPdfBusy(false);
  }

  const totalMinutes = sessions.reduce((a, s) => a + (Number(s.minutes) || 0), 0);
  const weekDays = lastNDays(7);
  const weekMinutes = sessions.filter((s) => weekDays.includes(s.date)).reduce((a, s) => a + s.minutes, 0);
  const avgSession = sessions.length ? Math.round(totalMinutes / sessions.length) : 0;

  const exerciseStats = sessions.reduce((acc, s) => {
    if (s.exerciseCount) {
      if (s.exerciseType === "تست") acc.tests += s.exerciseCount;
      else acc.exercises += s.exerciseCount;
    }
    return acc;
  }, { tests: 0, exercises: 0 });

  const topicBySubject = CURRICULUM.map((c) => {
    const subSessions = sessions.filter((s) => s.subjectId === c.id && s.topic);
    if (subSessions.length === 0) return null;
    const map = {};
    subSessions.forEach((s) => { map[s.topic] = (map[s.topic] || 0) + s.minutes; });
    const topics = Object.entries(map).map(([topic, minutes]) => ({ topic, minutes })).sort((a, b) => b.minutes - a.minutes);
    const total = topics.reduce((a, t) => a + t.minutes, 0);
    return { subject: c, topics, total };
  }).filter(Boolean).sort((a, b) => b.total - a.total);

  const perSubject = CURRICULUM
    .map((c) => ({
      name: c.name, color: c.color,
      hours: Math.round((sessions.filter((s) => s.subjectId === c.id).reduce((a, s) => a + s.minutes, 0) / 60) * 10) / 10,
    }))
    .filter((c) => c.hours > 0)
    .sort((a, b) => b.hours - a.hours);

  const perBox = boxes
    .map((b) => ({
      label: `باکس ${toFa(b.number)} (${toJalaliShort(new Date(b.dayKey + "T00:00:00"))})`,
      minutes: sessions.filter((s) => s.boxId === b.id).reduce((a, s) => a + s.minutes, 0),
    }))
    .filter((b) => b.minutes > 0)
    .sort((a, b) => b.minutes - a.minutes);

  const trend14 = lastNDays(14).map((dstr) => {
    const dateObj = new Date(dstr + "T00:00:00");
    const target = dayTargetMinutes(dateObj, settings, dayOverrides);
    const minutes = sessions.filter((s) => s.date === dstr).reduce((a, s) => a + s.minutes, 0);
    return { day: dayLabel(dstr), minutes, target, met: minutes >= target };
  });

  const currentWeek = getCurrentWeekDates();
  const currentWeekKeys = currentWeek.map((d) => todayKey(d));
  // سقف هفته = جمع اهداف روزانه خود کاربر (تنظیمات + استثناها)
  const weekGoalMinutes = currentWeek.reduce((sum, d) => sum + dayTargetMinutes(d, settings, dayOverrides), 0);
  const weekActualMinutes = sessions.filter((s) => currentWeekKeys.includes(s.date)).reduce((a, s) => a + s.minutes, 0);
  const weekGoalPct = Math.min(100, Math.round((weekActualMinutes / Math.max(1, weekGoalMinutes)) * 100));
  const todayKeyStr = todayKey();

  const weekBreakdown = currentWeek.map((d) => {
    const key = todayKey(d);
    const holiday = isHoliday(d);
    const target = dayTargetMinutes(d, settings, dayOverrides);
    const studied = sessions.filter((s) => s.date === key).reduce((a, s) => a + s.minutes, 0);
    const isFuture = key > todayKeyStr;
    return { date: d, key, holiday, target, studied, isFuture, met: studied >= target };
  });

  // ----- پیشنهاد برنامه هفته آینده بر اساس هفته گذشته -----
  const prevWeekDates = (() => {
    const start = new Date(currentWeek[0]);
    start.setDate(start.getDate() - 7);
    return Array.from({ length: 7 }, (_, i) => {
      const d = new Date(start);
      d.setDate(start.getDate() + i);
      return d;
    });
  })();
  const prevWeekKeys = prevWeekDates.map((d) => todayKey(d));
  const lastWeekMinutes = sessions
    .filter((s) => prevWeekKeys.includes(s.date))
    .reduce((a, s) => a + (Number(s.minutes) || 0), 0);
  const lastWeekBySubject = CURRICULUM.map((c) => {
    const mins = sessions
      .filter((s) => prevWeekKeys.includes(s.date) && s.subjectId === c.id)
      .reduce((a, s) => a + (Number(s.minutes) || 0), 0);
    return { ...c, minutes: mins };
  }).filter((c) => c.minutes > 0)
    .sort((a, b) => b.minutes - a.minutes);

  // دروس کم‌کار یا بدون مطالعه در ۲ هفته اخیر (اولویت جبرانی)
  const last14Keys = lastNDays(14);
  const weakSubjects = CURRICULUM
    .filter((c) => c.groupLabel !== "ویژه")
    .map((c) => {
      const mins = sessions
        .filter((s) => last14Keys.includes(s.date) && s.subjectId === c.id)
        .reduce((a, s) => a + (Number(s.minutes) || 0), 0);
      return { ...c, minutes: mins };
    })
    .sort((a, b) => a.minutes - b.minutes);

  const neglected = weakSubjects.filter((s) => s.minutes < 60).slice(0, 5);
  const topLastWeek = lastWeekBySubject.slice(0, 3);

  // پیشنهاد ساعت هفته بعد: اگر زیر هدف بود کمی بالاتر، اگر نزدیک بود حفظ، اگر بالاتر کمی معقول
  const gap = weekGoalMinutes - lastWeekMinutes;
  let suggestedWeekHours = WEEKLY_GOAL_HOURS;
  let planTone = "keep";
  if (lastWeekMinutes <= 0) {
    suggestedWeekHours = Math.min(WEEKLY_GOAL_HOURS, 25);
    planTone = "start";
  } else if (lastWeekMinutes < weekGoalMinutes * 0.6) {
    suggestedWeekHours = Math.round((lastWeekMinutes / 60) + 8);
    suggestedWeekHours = Math.min(WEEKLY_GOAL_HOURS, Math.max(20, suggestedWeekHours));
    planTone = "build";
  } else if (lastWeekMinutes < weekGoalMinutes) {
    suggestedWeekHours = WEEKLY_GOAL_HOURS;
    planTone = "push";
  } else {
    suggestedWeekHours = WEEKLY_GOAL_HOURS;
    planTone = "maintain";
  }

  const normalH = settings.normalHours ?? DEFAULT_NORMAL_DAY_HOURS;
  const holidayH = settings.holidayHours ?? DEFAULT_HOLIDAY_DAY_HOURS;
  // هفته آینده = ۷ روز بعد از شروع هفته جاری
  const nextWeekDates = currentWeek.map((d) => {
    const nd = new Date(d);
    nd.setDate(d.getDate() + 7);
    return nd;
  });
  const nextWeekPlan = nextWeekDates.map((d) => {
    const holiday = isHoliday(d);
    const hours = holiday ? holidayH : normalH;
    return {
      date: d,
      key: todayKey(d),
      holiday,
      hours,
      label: WEEKDAY_FULL_FA[d.getDay()],
      jalali: toJalaliShort(d),
    };
  });

  // تخصیص پیشنهادی دروس: ضعیف‌ها + ادامه قوی‌های هفته قبل
  const focusList = [];
  neglected.forEach((s) => {
    if (!focusList.find((x) => x.id === s.id)) focusList.push({ ...s, reason: "کم‌توجهی در دو هفته اخیر" });
  });
  topLastWeek.forEach((s) => {
    if (!focusList.find((x) => x.id === s.id)) focusList.push({ ...s, reason: "ادامه روند هفته قبل" });
  });
  // اگر هنوز کم است از عمومی/تخصصی پر کن
  if (focusList.length < 4) {
    CURRICULUM.filter((c) => c.groupLabel === "دوازدهم" || c.groupLabel === "عمومی").forEach((c) => {
      if (focusList.length >= 6) return;
      if (!focusList.find((x) => x.id === c.id)) focusList.push({ ...c, minutes: 0, reason: "پوشش پایه" });
    });
  }

  const planTips = [];
  if (planTone === "start") {
    planTips.push("هفته گذشته تقریباً مطالعه‌ای ثبت نشده؛ با هدف سبک‌تر شروع کن و هر روز حداقل یک باکس بساز.");
  } else if (planTone === "build") {
    planTips.push(`هفته گذشته ${formatDuration(lastWeekMinutes)} مطالعه کردی (زیر هدف). این هفته حدود ${toFa(suggestedWeekHours)} ساعت را هدف بگذار و روزانه را کمی بالاتر ببر.`);
  } else if (planTone === "push") {
    planTips.push(`تا هدف هفتگی ${toFa(Math.round(gap / 60))} ساعت فاصله داشتی. این هفته روی تکمیل هدف ${toFa(WEEKLY_GOAL_HOURS)} ساعته تمرکز کن.`);
  } else {
    planTips.push(`هفته گذشته خوب بودی (${formatDuration(lastWeekMinutes)}). همین ریتم را حفظ کن و دروس ضعیف را جبرانی کن.`);
  }
  if (neglected.length > 0) {
    planTips.push("درس‌هایی که کمتر خوانده‌ای را در باکس‌های جدا با زمان مشخص بگذار تا جبران شوند.");
  }
  if (topLastWeek.length > 0) {
    planTips.push("درس‌های پرکار هفته قبل را کامل قطع نکن؛ هر روز کمی مرور/تست برایشان بگذار.");
  }
  planTips.push("روزهای تعطیل (جمعه و یکشنبه) را برای آزمون جامع، تحلیل آزمون یا باکس جبرانی نگه دار.");

  if (sessions.length === 0) {
    return (
      <div className="kn-fade">
        <div style={{
          background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14,
          padding: 14, marginBottom: 16,
        }}>
          <div style={{ fontSize: 14, fontWeight: 800, marginBottom: 6 }}>گزارش PDF</div>
          <div style={{ fontSize: 11.5, color: COLORS.muted, marginBottom: 12, lineHeight: 1.7 }}>
            هنوز جلسه‌ای ثبت نشده؛ بعد از ثبت مطالعه می‌توانی گزارش بگیری.
          </div>
          <div style={{ fontSize: 11, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>از تاریخ</div>
          <div style={{ marginBottom: 10 }}><JalaliDatePicker value={reportFrom} onChange={setReportFrom} /></div>
          <div style={{ fontSize: 11, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>تا تاریخ</div>
          <div style={{ marginBottom: 12 }}><JalaliDatePicker value={reportTo} onChange={setReportTo} /></div>
          <button onClick={handleDownloadPdf} disabled={pdfBusy} className="kn-btn" style={{
            width: "100%", background: COLORS.accent, color: "#1a1400", border: "none",
            borderRadius: 10, padding: "11px 0", fontSize: 13.5, fontWeight: 800,
          }}>{pdfBusy ? "در حال ساخت PDF..." : "دانلود گزارش PDF"}</button>
          {pdfMsg && <div style={{ fontSize: 12, color: COLORS.accentSoft, marginTop: 10, textAlign: "center" }}>{pdfMsg}</div>}
        </div>
        <div style={{ textAlign: "center", padding: "40px 20px", color: COLORS.muted }}>
          <div style={{ fontSize: 14 }}>وقتی چند جلسه مطالعه ثبت کنی، تحلیل عملکردت اینجا نمایش داده می‌شود.</div>
        </div>
      </div>
    );
  }

  return (
    <div className="kn-fade">
      {/* stat cards */}
      <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: 10, marginBottom: 18 }}>
        <StatCard icon={Clock3} label="مجموع کل" value={formatDuration(totalMinutes)} />
        <StatCard icon={Box} label="این هفته" value={formatDuration(weekMinutes)} />
        <StatCard icon={Flame} label="روزهای متوالی" value={`${toFa(streak)} روز`} />
        <StatCard icon={CheckCircle2} label="تعداد جلسات" value={toFa(sessions.length)} />
        <StatCard icon={Clock3} label="میانگین هر جلسه" value={`${toFa(avgSession)} دقیقه`} />
        <StatCard icon={CheckCircle2} label="تست / تمرین حل‌شده" value={`${toFa(exerciseStats.tests)} / ${toFa(exerciseStats.exercises)}`} />
      </div>

      {/* جزئیات یک روز خاص */}
      {(() => {
        const dObj = new Date(detailDate + "T00:00:00");
        const daySessions = sessions.filter((s) => s.date === detailDate).sort((a, b) => (b.timestamp || 0) - (a.timestamp || 0));
        const dayBoxes = boxes.filter((b) => b.dayKey === detailDate).sort((a, b) => (a.number || 0) - (b.number || 0));
        const dayMin = daySessions.reduce((a, s) => a + (Number(s.minutes) || 0), 0);
        const dayTarget = dayTargetMinutes(dObj, settings, dayOverrides);
        const dayPct = Math.min(100, Math.round((dayMin / Math.max(1, dayTarget)) * 100));
        const dayTests = daySessions.reduce((a, s) => a + (s.exerciseType === "تست" ? (s.exerciseCount || 0) : 0), 0);
        const dayEx = daySessions.reduce((a, s) => a + (s.exerciseType !== "تست" ? (s.exerciseCount || 0) : 0), 0);
        const bySubject = {};
        daySessions.forEach((s) => {
          const sub = curriculumById(s.subjectId);
          const name = sub?.name || "نامشخص";
          if (!bySubject[name]) bySubject[name] = { minutes: 0, color: sub?.color || COLORS.muted };
          bySubject[name].minutes += Number(s.minutes) || 0;
        });
        const subjectRows = Object.entries(bySubject).sort((a, b) => b[1].minutes - a[1].minutes);
        return (
          <div style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14, padding: 14, marginBottom: 16 }}>
            <div style={{ fontSize: 13, fontWeight: 700, marginBottom: 8, color: COLORS.muted }}>جزئیات روز</div>
            <div style={{ marginBottom: 10 }}>
              <JalaliDatePicker value={detailDate} onChange={setDetailDate} />
            </div>
            <div style={{ display: "flex", flexWrap: "wrap", gap: 8, marginBottom: 10 }}>
              <span style={{ fontSize: 12, fontWeight: 700 }}>{WEEKDAY_FULL_FA[dObj.getDay()]} · {toJalaliStr(dObj)}</span>
              <span style={{
                fontSize: 10.5, fontWeight: 700, padding: "2px 8px", borderRadius: 999,
                background: isHoliday(dObj) ? COLORS.accent + "22" : COLORS.accent2 + "22",
                color: isHoliday(dObj) ? COLORS.accent : COLORS.accent2,
              }}>{isHoliday(dObj) ? "تعطیل" : "عادی"}</span>
            </div>
            <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: 8, marginBottom: 10 }}>
              <div style={{ background: COLORS.bg, borderRadius: 10, padding: "8px 10px" }}>
                <div style={{ fontSize: 10.5, color: COLORS.muted }}>مطالعه</div>
                <div style={{ fontSize: 13, fontWeight: 800 }}>{formatDuration(dayMin)}</div>
              </div>
              <div style={{ background: COLORS.bg, borderRadius: 10, padding: "8px 10px" }}>
                <div style={{ fontSize: 10.5, color: COLORS.muted }}>هدف روز</div>
                <div style={{ fontSize: 13, fontWeight: 800 }}>{formatDuration(dayTarget)} · {toFa(dayPct)}٪</div>
              </div>
              <div style={{ background: COLORS.bg, borderRadius: 10, padding: "8px 10px" }}>
                <div style={{ fontSize: 10.5, color: COLORS.muted }}>جلسات / باکس</div>
                <div style={{ fontSize: 13, fontWeight: 800 }}>{toFa(daySessions.length)} / {toFa(dayBoxes.length)}</div>
              </div>
              <div style={{ background: COLORS.bg, borderRadius: 10, padding: "8px 10px" }}>
                <div style={{ fontSize: 10.5, color: COLORS.muted }}>تست / تمرین</div>
                <div style={{ fontSize: 13, fontWeight: 800 }}>{toFa(dayTests)} / {toFa(dayEx)}</div>
              </div>
            </div>
            <div style={{ height: 7, borderRadius: 99, background: COLORS.surface2, overflow: "hidden", marginBottom: 12 }}>
              <div style={{
                height: "100%", width: `${dayPct}%`, borderRadius: 99,
                background: dayPct >= 100 ? COLORS.accent2 : COLORS.accent,
              }} />
            </div>

            <div style={{ fontSize: 12, fontWeight: 700, color: COLORS.muted, marginBottom: 6 }}>باکس‌های آن روز</div>
            {dayBoxes.length === 0 ? (
              <div style={{ fontSize: 12, color: COLORS.muted, marginBottom: 10 }}>باکسی ثبت نشده</div>
            ) : (
              <div style={{ display: "flex", flexDirection: "column", gap: 6, marginBottom: 12 }}>
                {dayBoxes.map((b) => {
                  const mins = sessions.filter((s) => s.boxId === b.id).reduce((a, s) => a + (Number(s.minutes) || 0), 0);
                  const names = (b.subjectIds || []).map((id) => curriculumById(id)?.name).filter(Boolean).join("، ");
                  return (
                    <div key={b.id} style={{ background: COLORS.bg, borderRadius: 10, padding: "8px 10px" }}>
                      <div style={{ fontSize: 12.5, fontWeight: 700 }}>باکس {toFa(b.number)} · {formatDuration(mins)}</div>
                      <div style={{ fontSize: 11, color: COLORS.muted, marginTop: 2 }}>{names || "بدون درس"}</div>
                    </div>
                  );
                })}
              </div>
            )}

            <div style={{ fontSize: 12, fontWeight: 700, color: COLORS.muted, marginBottom: 6 }}>تفکیک دروس</div>
            {subjectRows.length === 0 ? (
              <div style={{ fontSize: 12, color: COLORS.muted, marginBottom: 10 }}>مطالعه‌ای نیست</div>
            ) : (
              <div style={{ display: "flex", flexDirection: "column", gap: 6, marginBottom: 12 }}>
                {subjectRows.map(([name, info]) => (
                  <div key={name} style={{ display: "flex", alignItems: "center", justifyContent: "space-between" }}>
                    <span style={{ display: "flex", alignItems: "center", gap: 6, fontSize: 12.5 }}>
                      <span style={{ width: 7, height: 7, borderRadius: 99, background: info.color }} />
                      {name}
                    </span>
                    <span style={{ fontSize: 12, color: COLORS.muted, fontWeight: 600 }}>{formatDuration(info.minutes)}</span>
                  </div>
                ))}
              </div>
            )}

            <div style={{ fontSize: 12, fontWeight: 700, color: COLORS.muted, marginBottom: 6 }}>جلسات مطالعه</div>
            {daySessions.length === 0 ? (
              <div style={{ fontSize: 12, color: COLORS.muted }}>جلسه‌ای ثبت نشده</div>
            ) : (
              <div style={{ display: "flex", flexDirection: "column", gap: 6 }}>
                {daySessions.map((s) => {
                  const sub = curriculumById(s.subjectId);
                  return (
                    <div key={s.id} style={{ background: COLORS.bg, borderRadius: 10, padding: "8px 10px" }}>
                      <div style={{ display: "flex", justifyContent: "space-between", gap: 8 }}>
                        <span style={{ fontSize: 12.5, fontWeight: 700 }}>{sub?.name || "درس"}</span>
                        <span style={{ fontSize: 12, color: COLORS.muted }}>{formatDuration(s.minutes)}</span>
                      </div>
                      {(s.topic || s.note) && (
                        <div style={{ fontSize: 11, color: COLORS.muted, marginTop: 3 }}>
                          {s.topic ? s.topic : ""}{s.topic && s.note ? " · " : ""}{s.note || ""}
                        </div>
                      )}
                      {s.exerciseCount ? (
                        <div style={{ fontSize: 10.5, color: COLORS.accentSoft, marginTop: 3 }}>
                          {s.exerciseType || "تمرین"}: {toFa(s.exerciseCount)}
                        </div>
                      ) : null}
                    </div>
                  );
                })}
              </div>
            )}
          </div>
        );
      })()}

      {/* weekly goal (۴۳ ساعت) */}
      <div style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14, padding: 14, marginBottom: 16 }}>
        <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 8 }}>
          <div style={{ fontSize: 13, fontWeight: 700, color: COLORS.muted }}>هدف این هفته (شنبه تا جمعه)</div>
          <div style={{ fontSize: 12.5, fontWeight: 700, color: weekGoalPct >= 100 ? COLORS.accent2 : COLORS.text }}>
            {formatDuration(weekActualMinutes)} از {formatDuration(weekGoalMinutes)}
          </div>
        </div>
        <div style={{ height: 9, borderRadius: 99, background: COLORS.surface2, overflow: "hidden" }}>
          <div style={{
            height: "100%", width: `${weekGoalPct}%`, borderRadius: 99,
            background: weekGoalPct >= 100 ? COLORS.accent2 : COLORS.accent, transition: "width .4s ease",
          }} />
        </div>
        <div style={{ fontSize: 11, color: COLORS.muted, marginTop: 8 }}>
          سقف این هفته بر اساس هدف‌هایی است که خودت تعیین کردی
          (روز عادی: {formatDuration((settings.normalHours ?? DEFAULT_NORMAL_DAY_HOURS) * 60)}،
          {" "}روز تعطیل: {formatDuration((settings.holidayHours ?? DEFAULT_HOLIDAY_DAY_HOURS) * 60)}
          {Object.keys(dayOverrides || {}).some((k) => currentWeekKeys.includes(k)) ? " · به‌همراه استثناهای این هفته" : ""}).
        </div>

        <div style={{ display: "flex", flexDirection: "column", gap: 6, marginTop: 14 }}>
          {weekBreakdown.map((row) => (
            <div key={row.key} style={{
              display: "flex", alignItems: "center", justifyContent: "space-between",
              padding: "7px 10px", borderRadius: 10,
              background: row.key === todayKeyStr ? COLORS.surface2 : "transparent",
              opacity: row.isFuture ? 0.55 : 1,
            }}>
              <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
                <span style={{
                  fontSize: 10.5, fontWeight: 700, padding: "2px 7px", borderRadius: 999,
                  background: row.holiday ? COLORS.accent + "22" : COLORS.accent2 + "22",
                  color: row.holiday ? COLORS.accent : COLORS.accent2, minWidth: 40, textAlign: "center",
                }}>{row.holiday ? "تعطیل" : "عادی"}</span>
                <div>
                  <div style={{ fontSize: 12.5, fontWeight: 600 }}>{WEEKDAY_FULL_FA[row.date.getDay()]}</div>
                  <div style={{ fontSize: 10.5, color: COLORS.muted }}>{toJalaliShort(row.date)}</div>
                </div>
              </div>
              <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
                <span style={{ fontSize: 11.5, color: COLORS.muted }}>
                  {row.isFuture ? "—" : formatDuration(row.studied)} / {formatDuration(row.target)}
                </span>
                {!row.isFuture && (row.met
                  ? <CheckCircle2 size={15} color={COLORS.accent2} />
                  : <Circle size={15} color={COLORS.muted} />)}
              </div>
            </div>
          ))}
        </div>
      </div>

      {/* weekly trend - CSS bars (بدون وابستگی به Recharts) */}
      <div style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14, padding: 14, marginBottom: 16 }}>
        <div style={{ fontSize: 13, fontWeight: 700, marginBottom: 10, color: COLORS.muted }}>روند ۱۴ روز اخیر در برابر هدف روزانه</div>
        <div style={{ display: "flex", alignItems: "flex-end", gap: 4, height: 140, paddingTop: 8 }}>
          {trend14.map((e, i) => {
            const maxM = Math.max(1, ...trend14.map((x) => Math.max(x.minutes, x.target)));
            const h = Math.round((e.minutes / maxM) * 100);
            return (
              <div key={i} style={{ flex: 1, display: "flex", flexDirection: "column", alignItems: "center", height: "100%", justifyContent: "flex-end" }}>
                <div style={{
                  width: "100%", maxWidth: 18, height: `${Math.max(h, e.minutes > 0 ? 4 : 0)}%`,
                  background: e.met ? COLORS.accent2 : COLORS.accent, borderRadius: "4px 4px 0 0",
                }} title={`${e.minutes} دقیقه`} />
                <div style={{ fontSize: 9, color: COLORS.muted, marginTop: 4 }}>{e.day}</div>
              </div>
            );
          })}
        </div>
        <div style={{ display: "flex", gap: 14, marginTop: 10, fontSize: 10.5, color: COLORS.muted }}>
          <span><span style={{ display: "inline-block", width: 8, height: 8, borderRadius: 2, background: COLORS.accent2, marginLeft: 4 }} />هدف محقق‌شده</span>
          <span><span style={{ display: "inline-block", width: 8, height: 8, borderRadius: 2, background: COLORS.accent, marginLeft: 4 }} />کمتر از هدف</span>
        </div>
      </div>

      {/* per subject */}
      {perSubject.length > 0 && (
        <div style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14, padding: 14, marginBottom: 16 }}>
          <div style={{ fontSize: 13, fontWeight: 700, marginBottom: 10, color: COLORS.muted }}>ساعت مطالعه به تفکیک درس</div>
          <div style={{ display: "flex", flexDirection: "column", gap: 10 }}>
            {perSubject.map((entry) => {
              const maxH = Math.max(0.1, ...perSubject.map((x) => x.hours));
              const pct = Math.round((entry.hours / maxH) * 100);
              return (
                <div key={entry.name}>
                  <div style={{ display: "flex", justifyContent: "space-between", marginBottom: 4 }}>
                    <span style={{ fontSize: 12.5, fontWeight: 600 }}>{entry.name}</span>
                    <span style={{ fontSize: 11.5, color: COLORS.muted }}>{toFa(entry.hours)} ساعت</span>
                  </div>
                  <div style={{ height: 8, borderRadius: 99, background: COLORS.surface2, overflow: "hidden" }}>
                    <div style={{ width: `${pct}%`, height: "100%", background: entry.color, borderRadius: 99 }} />
                  </div>
                </div>
              );
            })}
          </div>
        </div>
      )}

      {/* topics studied per subject */}
      {topicBySubject.length > 0 && (
        <div style={{ marginBottom: 16 }}>
          <div style={{ fontSize: 13, fontWeight: 700, marginBottom: 10, color: COLORS.muted }}>مباحث مطالعه‌شده</div>
          <div style={{ display: "flex", flexDirection: "column", gap: 10 }}>
            {topicBySubject.map(({ subject, topics, total }) => {
              const maxT = Math.max(1, ...topics.map((t) => t.minutes));
              return (
                <div key={subject.id} style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14, padding: 14 }}>
                  <div style={{ display: "flex", alignItems: "center", gap: 8, marginBottom: 8 }}>
                    <span style={{ width: 8, height: 8, borderRadius: 99, background: subject.color }} />
                    <span style={{ fontSize: 13, fontWeight: 700 }}>{subject.name}</span>
                    <span style={{ fontSize: 11, color: COLORS.muted, marginRight: "auto" }}>{formatDuration(total)}</span>
                  </div>
                  <div style={{ display: "flex", flexDirection: "column", gap: 6 }}>
                    {topics.map((t) => (
                      <div key={t.topic} style={{ display: "flex", alignItems: "center", gap: 8 }}>
                        <span style={{ fontSize: 11.5, color: COLORS.muted, flex: 1, minWidth: 0, overflow: "hidden", textOverflow: "ellipsis", whiteSpace: "nowrap" }}>{t.topic}</span>
                        <div style={{ width: 70, height: 6, borderRadius: 99, background: COLORS.surface2, overflow: "hidden", flexShrink: 0 }}>
                          <div style={{ width: `${Math.round((t.minutes / maxT) * 100)}%`, height: "100%", background: subject.color, borderRadius: 99 }} />
                        </div>
                        <span style={{ fontSize: 10.5, color: COLORS.muted, width: 54, textAlign: "left", flexShrink: 0 }}>{formatDuration(t.minutes)}</span>
                      </div>
                    ))}
                  </div>
                </div>
              );
            })}
          </div>
        </div>
      )}

      {/* per box */}
      {perBox.length > 0 && (
        <div style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14, padding: 14, marginBottom: 16 }}>
          <div style={{ fontSize: 13, fontWeight: 700, marginBottom: 10, color: COLORS.muted }}>زمان به تفکیک باکس</div>
          <div style={{ display: "flex", flexDirection: "column", gap: 8 }}>
            {perBox.map((b) => (
              <div key={b.label} style={{ display: "flex", alignItems: "center", justifyContent: "space-between" }}>
                <span style={{ fontSize: 12.5 }}>{b.label}</span>
                <span style={{ fontSize: 12, color: COLORS.muted, fontWeight: 600 }}>{formatDuration(b.minutes)}</span>
              </div>
            ))}
          </div>
        </div>
      )}

      {/* پیشنهاد برنامه هفته آینده */}
      <div style={{ background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14, padding: 14, marginBottom: 8 }}>
        <div style={{ fontSize: 14, fontWeight: 800, marginBottom: 4, display: "flex", alignItems: "center", gap: 6 }}>
          <Trophy size={16} color={COLORS.accent} /> پیشنهاد برنامه هفته آینده
        </div>
        <div style={{ fontSize: 11.5, color: COLORS.muted, marginBottom: 12, lineHeight: 1.7 }}>
          بر اساس مطالعه هفته گذشته ({formatDuration(lastWeekMinutes)} از {toFa(WEEKLY_GOAL_HOURS)} ساعت هدف)
        </div>

        <div style={{
          background: COLORS.surface2, borderRadius: 12, padding: "10px 12px", marginBottom: 12,
          border: `1px solid ${COLORS.border}`,
        }}>
          <div style={{ fontSize: 12, color: COLORS.muted, marginBottom: 4 }}>هدف پیشنهادی هفته بعد</div>
          <div style={{ fontSize: 18, fontWeight: 800, color: COLORS.accentSoft }}>{toFa(suggestedWeekHours)} ساعت</div>
        </div>

        <div style={{ fontSize: 12.5, fontWeight: 700, color: COLORS.muted, marginBottom: 8 }}>برنامه روزانه پیشنهادی</div>
        <div style={{ display: "flex", flexDirection: "column", gap: 6, marginBottom: 14 }}>
          {nextWeekPlan.map((row) => (
            <div key={row.key} style={{
              display: "flex", alignItems: "center", justifyContent: "space-between",
              padding: "8px 10px", borderRadius: 10, background: COLORS.bg,
            }}>
              <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
                <span style={{
                  fontSize: 10, fontWeight: 700, padding: "2px 7px", borderRadius: 999,
                  background: row.holiday ? COLORS.accent + "22" : COLORS.accent2 + "22",
                  color: row.holiday ? COLORS.accent : COLORS.accent2,
                }}>{row.holiday ? "تعطیل" : "عادی"}</span>
                <div>
                  <div style={{ fontSize: 12.5, fontWeight: 600 }}>{row.label}</div>
                  <div style={{ fontSize: 10.5, color: COLORS.muted }}>{row.jalali}</div>
                </div>
              </div>
              <span style={{ fontSize: 12, fontWeight: 700 }}>{formatDuration(row.hours * 60)}</span>
            </div>
          ))}
        </div>

        <div style={{ fontSize: 12.5, fontWeight: 700, color: COLORS.muted, marginBottom: 8 }}>درس‌های پیشنهادی برای تمرکز</div>
        <div style={{ display: "flex", flexDirection: "column", gap: 8, marginBottom: 14 }}>
          {focusList.slice(0, 6).map((s) => (
            <div key={s.id} style={{
              display: "flex", alignItems: "flex-start", gap: 10,
              padding: "10px 12px", borderRadius: 10, background: COLORS.bg,
              border: `1px solid ${COLORS.border}`,
            }}>
              <span style={{ width: 8, height: 8, borderRadius: 99, background: s.color, marginTop: 5, flexShrink: 0 }} />
              <div style={{ flex: 1, minWidth: 0 }}>
                <div style={{ fontSize: 13, fontWeight: 700 }}>{s.name}</div>
                <div style={{ fontSize: 11, color: COLORS.muted, marginTop: 2 }}>{s.reason}</div>
                {s.minutes > 0 && (
                  <div style={{ fontSize: 10.5, color: COLORS.accentSoft, marginTop: 2 }}>
                    هفته قبل: {formatDuration(s.minutes)}
                  </div>
                )}
              </div>
            </div>
          ))}
        </div>

        <div style={{ fontSize: 12.5, fontWeight: 700, color: COLORS.muted, marginBottom: 8 }}>نکته‌های اجرایی</div>
        <div style={{ display: "flex", flexDirection: "column", gap: 6 }}>
          {planTips.map((tip, i) => (
            <div key={i} style={{
              fontSize: 12, color: COLORS.text, lineHeight: 1.8,
              padding: "8px 10px", borderRadius: 10, background: COLORS.bg,
            }}>
              <span style={{ color: COLORS.accent, fontWeight: 700 }}>{toFa(i + 1)}. </span>
              {tip}
            </div>
          ))}
        </div>
      </div>

      {/* گزارش PDF */}
      <div style={{
        background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14,
        padding: 14, marginTop: 8, marginBottom: 8,
      }}>
        <div style={{ fontSize: 14, fontWeight: 800, marginBottom: 6, display: "flex", alignItems: "center", gap: 6 }}>
          گزارش PDF
        </div>
        <div style={{ fontSize: 11.5, color: COLORS.muted, marginBottom: 12, lineHeight: 1.7 }}>
          بازه زمانی را انتخاب کن و گزارش مطالعه (خلاصه، تفکیک درس و روز، جزئیات جلسات) را به‌صورت PDF بگیر.
        </div>
        <div style={{ fontSize: 11, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>از تاریخ</div>
        <div style={{ marginBottom: 10 }}>
          <JalaliDatePicker value={reportFrom} onChange={setReportFrom} />
        </div>
        <div style={{ fontSize: 11, color: COLORS.muted, fontWeight: 700, marginBottom: 6 }}>تا تاریخ</div>
        <div style={{ marginBottom: 12 }}>
          <JalaliDatePicker value={reportTo} onChange={setReportTo} />
        </div>
        <button onClick={handleDownloadPdf} disabled={pdfBusy} className="kn-btn" style={{
          width: "100%", background: COLORS.accent, color: "#1a1400", border: "none",
          borderRadius: 10, padding: "11px 0", fontSize: 13.5, fontWeight: 800,
        }}>
          {pdfBusy ? "در حال ساخت PDF..." : "دانلود گزارش PDF"}
        </button>
        {pdfMsg && (
          <div style={{ fontSize: 12, color: COLORS.accentSoft, marginTop: 10, textAlign: "center" }}>{pdfMsg}</div>
        )}
      </div>

    </div>
  );
}

function StatCard({ icon: Icon, label, value }) {
  return (
    <div style={{
      background: COLORS.surface, border: `1px solid ${COLORS.border}`, borderRadius: 14, padding: "12px 14px",
    }}>
      <div style={{ display: "flex", alignItems: "center", gap: 6, color: COLORS.muted, fontSize: 11.5, marginBottom: 6 }}>
        <Icon size={13} /> {label}
      </div>
      <div style={{ fontSize: 16, fontWeight: 800 }}>{value}</div>
    </div>
  );
}

    const root = ReactDOM.createRoot(document.getElementById("root"));
    root.render(React.createElement(App));
  </script>
</body>
</html>
