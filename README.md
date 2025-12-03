// ===============================
// FULL COMBINED FIREBASE + REACT WATCH-AND-EARN SYSTEM
// with Firestore availability detection and in-memory fallback
// ===============================

import React, { useState, useEffect, useRef } from 'react';

// -------------------------------
// Firebase SDK imports (modular)
// -------------------------------
import { initializeApp } from 'firebase/app';
// We import Firestore functions but will guard usage at runtime
import {
  getFirestore,
  doc,
  getDoc,
  setDoc,
  updateDoc,
  increment
} from 'firebase/firestore';

// -------------------------------
// Firebase config (REPLACE with your values)
// -------------------------------
const firebaseConfig = {
  apiKey: 'YOUR_API_KEY',
  authDomain: 'YOUR_AUTH_DOMAIN',
  projectId: 'YOUR_PROJECT_ID',
  storageBucket: 'YOUR_BUCKET',
  messagingSenderId: 'YOUR_SENDER_ID',
  appId: 'YOUR_APP_ID'
};

// initialize app
let app = null;
let db = null;
let FIRESTORE_AVAILABLE = false;
try {
  app = initializeApp(firebaseConfig);
  // Attempt to get Firestore - this can throw in restricted environments
  db = getFirestore(app);
  FIRESTORE_AVAILABLE = !!db;
  console.log('Firestore initialized:', FIRESTORE_AVAILABLE);
} catch (err) {
  FIRESTORE_AVAILABLE = false;
  console.warn('Firestore not available in this environment — falling back to in-memory DB. Error:', err && err.message);
}

// -------------------------------
// In-memory fallback DB (used when Firestore unavailable)
// -------------------------------
const inMemoryDB = {
  users: new Map()
};

// -------------------------------
// Unified FIRE API — uses Firestore when available, otherwise in-memory
// Methods: getUser(uid), createUser(uid,data), creditUser(uid,points,meta), requestPayout(uid,amount)
// -------------------------------
export const FIRE = {
  async getUser(uid) {
    if (!uid) return null;
    if (FIRESTORE_AVAILABLE) {
      const ref = doc(db, 'users', uid);
      const snap = await getDoc(ref);
      return snap.exists() ? snap.data() : null;
    } else {
      const data = inMemoryDB.users.get(uid) || null;
      return data;
    }
  },

  async createUser(uid, data = {}) {
    if (!uid) throw new Error('uid-required');
    const record = { id: uid, points: 0, email: data.email || null, createdAt: Date.now(), ...data };
    if (FIRESTORE_AVAILABLE) {
      await setDoc(doc(db, 'users', uid), record);
      return true;
    } else {
      inMemoryDB.users.set(uid, record);
      return true;
    }
  },

  async creditUser(uid, points, meta = {}) {
    if (!uid) throw new Error('uid-required');
    if (typeof points !== 'number') points = Number(points) || 0;
    if (FIRESTORE_AVAILABLE) {
      const userRef = doc(db, 'users', uid);
      // updateDoc with increment is atomic in Firestore
      await updateDoc(userRef, {
        points: increment(points),
        lastReward: meta || null
      });
      const snap = await getDoc(userRef);
      return snap.exists() ? snap.data().points : null;
    } else {
      const existing = inMemoryDB.users.get(uid) || { points: 0, id: uid };
      existing.points = (existing.points || 0) + points;
      existing.lastReward = meta || null;
      inMemoryDB.users.set(uid, existing);
      return existing.points;
    }
  },

  async requestPayout(uid, amount, method = 'paypal') {
    if (!uid) throw new Error('uid-required');
    if (FIRESTORE_AVAILABLE) {
      const userRef = doc(db, 'users', uid);
      // In production use a transaction to avoid race conditions
      await updateDoc(userRef, { points: 0 });
      const payoutRef = doc(userRef, 'payouts', Math.random().toString(36).slice(2));
      // setDoc for subcollection entry
      await setDoc(payoutRef, { amount, method, status: 'pending', requestedAt: new Date().toISOString() });
      return true;
    } else {
      const user = inMemoryDB.users.get(uid);
      if (!user) throw new Error('user-not-found');
      user.points = 0;
      user.payouts = user.payouts || [];
      user.payouts.push({ id: Math.random().toString(36).slice(2), amount, method, status: 'pending', requestedAt: new Date().toISOString() });
      inMemoryDB.users.set(uid, user);
      return true;
    }
  }
};

// -------------------------------
// Rewarding watched videos
// -------------------------------
const POINTS_PER_MINUTE = 10;

async function rewardUserFirestore(uid, video) {
  if (!uid) return;
  const pointsEarned = Math.max(1, Math.round((video.duration / 60) * POINTS_PER_MINUTE));
  try {
    const newPoints = await FIRE.creditUser(uid, pointsEarned, { source: 'video', videoId: video.id, title: video.title });
    console.log(`Credited ${pointsEarned} points to ${uid}. New total: ${newPoints}`);
  } catch (err) {
    console.error('Error crediting user:', err && err.message);
  }
}

function startWatchFirestore(user, video, setWatching, setMessage) {
  if (!user) {
    setMessage('Please log in to watch and earn.');
    return;
  }
  setWatching({ video, remaining: video.duration });
  setMessage('Watching: ' + video.title);
}

// -------------------------------
// React App — includes fallback login (mock) and uses FIRE API
// -------------------------------
export default function App() {
  // For demo purposes we provide a simple mock login flow.
  // In production wire Firebase Auth and call FIRE.createUser on signup.
  const [uid, setUid] = useState(null);
  const [watching, setWatching] = useState(null);
  const [message, setMessage] = useState('');
  const [points, setPoints] = useState(0);
  const timerRef = useRef(null);

  // Demo videos
  const videos = [
    { id: 'v1', title: 'Ad — 15s', duration: 15 },
    { id: 'v2', title: 'Promo — 30s', duration: 30 },
    { id: 'v3', title: 'Trailer — 45s', duration: 45 }
  ];

  // if uid changes, load user profile (or create)
  useEffect(() => {
    let cancelled = false;
    async function load() {
      if (!uid) return;
      const data = await FIRE.getUser(uid);
      if (!data) {
        await FIRE.createUser(uid, { email: null });
        setPoints(0);
      } else {
        if (!cancelled) setPoints(data.points || 0);
      }
    }
    load();
    return () => { cancelled = true; };
  }, [uid]);

  // Listen for changes in Firestore when available (real-time). For in-memory fallback we poll.
  useEffect(() => {
    if (!uid) return;
    if (FIRESTORE_AVAILABLE) {
      // Real-time listener
      try {
        const unsub = (function subscribe() {
          const userRef = doc(db, 'users', uid);
          // onSnapshot is optional import; to avoid adding another import, use a simple polling fallback here
          // If you want real-time updates, import onSnapshot from 'firebase/firestore' and use it.
          return () => {};
        })();
        return unsub;
      } catch (err) {
        console.warn('Realtime listener setup failed, falling back to polling:', err && err.message);
      }
    }
    // Polling fallback every 3s for in-memory or when realtime not set
    let stopped = false;
    const poll = async () => {
      while (!stopped) {
        const u = await FIRE.getUser(uid);
        if (u) setPoints(u.points || 0);
        // small delay
        await new Promise(r => setTimeout(r, 3000));
      }
    };
    poll();
    return () => { stopped = true; };
  }, [uid]);

  // watch timer
  useEffect(() => {
    if (!watching) return;
    timerRef.current = setInterval(() => {
      setWatching(prev => {
        if (!prev) return null;
        if (prev.remaining <= 1) {
          clearInterval(timerRef.current);
          // credit
          rewardUserFirestore(uid, prev.video);
          setMessage(`Finished: ${prev.video.title}`);
          return null;
        }
        return { ...prev, remaining: prev.remaining - 1 };
      });
    }, 1000);
    return () => clearInterval(timerRef.current);
  }, [watching, uid]);

  // simple mock login for demo
  const handleMockLogin = async () => {
    const demoUid = 'demo_' + Math.random().toString(36).slice(2,8);
    setUid(demoUid);
    await FIRE.createUser(demoUid, { email: null });
    setMessage('Logged in as demo user: ' + demoUid);
  };

  const handleRequestPayout = async () => {
    if (!uid) { setMessage('Login first'); return; }
    try {
      await FIRE.requestPayout(uid, Math.floor((points || 0) / 100));
      setMessage('Payout requested');
      setPoints(0);
    } catch (err) {
      setMessage('Payout failed: ' + (err && err.message));
    }
  };

  return (
    <div style={{ padding: 20, fontFamily: 'Arial, Helvetica, sans-serif' }}>
      <h1>VideoCash — Watch & Earn</h1>
      <div style={{ marginBottom: 12 }}>
        {uid ? (
          <div>
            <strong>User:</strong> {uid} &nbsp; <button onClick={() => { setUid(null); setMessage('Logged out'); }}>Logout</button>
          </div>
        ) : (
          <div>
            <button onClick={handleMockLogin}>Login (demo)</button>
            <span style={{ marginLeft: 8, color: '#666' }}>or connect real Firebase Auth in production</span>
          </div>
        )}
      </div>

      <div style={{ marginBottom: 12 }}><strong>Points:</strong> {points}</div>
      <div style={{ marginBottom: 12 }}>{message}</div>

      <h2>Videos</h2>
      <div style={{ display: 'grid', gap: 8 }}>
        {videos.map(v => (
          <div key={v.id} style={{ padding: 8, border: '1px solid #eee', borderRadius: 6 }}>
            <div style={{ fontWeight: 600 }}>{v.title}</div>
            <div style={{ fontSize: 12, color: '#666' }}>Duration: {v.duration}s</div>
            <div style={{ marginTop: 6 }}>
              <button onClick={() => startWatchFirestore({ uid }, v, setWatching, setMessage)}>Watch & Earn</button>
            </div>
          </div>
        ))}
      </div>

      {watching && (
        <div style={{ marginTop: 16 }}>
          <h3>Watching: {watching.video.title}</h3>
          <p>Time left: {watching.remaining}s</p>
        </div>
      )}

      <div style={{ marginTop: 20 }}>
        <button onClick={handleRequestPayout}>Request Payout</button>
      </div>

      <div style={{ marginTop: 20, color: '#666' }}>
        <p>Firestore available: {FIRESTORE_AVAILABLE ? 'Yes' : 'No (using in-memory fallback)'}</p>
      </div>
    </div>
  );
}
