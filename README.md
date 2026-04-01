import React, { useState, useEffect, useRef } from 'react';
import { 
  ChevronLeft, 
  ChevronRight, 
  Plus, 
  Trash2, 
  Settings,
  Clock, 
  Play, 
  Pause, 
  RotateCcw,
  Cloud,
  Check,
  Star,
  Moon,
  Wind
} from 'lucide-react';

// --- Firebase Imports ---
import { initializeApp } from 'firebase/app';
import { getAuth, signInWithCustomToken, signInAnonymously, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, onSnapshot, collection } from 'firebase/firestore';

// ==========================================================
// FIREBASE CONFIGURATION
// ==========================================================
const firebaseConfig = {
  apiKey: "AIzaSyBv14g9crV8vGbobK5cdVxwqlrr0EiM0fA",
  authDomain: "bokk-55eae.firebaseapp.com",
  projectId: "bokk-55eae",
  storageBucket: "bokk-55eae.firebasestorage.app",
  messagingSenderId: "757990079253",
  appId: "1:757990079253:web:f3dbf171b2137f436bc71c",
  measurementId: "G-2PMDNT4LM2"
};

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const APP_ID = typeof __app_id !== 'undefined' ? __app_id : 'planner-app-v1';

const App = () => {
  const [user, setUser] = useState(null);
  const [view, setView] = useState('daily');
  const [records, setRecords] = useState({});
  const [loading, setLoading] = useState(true);
  const [dDayTarget, setDDayTarget] = useState('');

  const getTodayStr = () => {
    const d = new Date();
    return `${d.getFullYear()}${String(d.getMonth() + 1).padStart(2, '0')}${String(d.getDate()).padStart(2, '0')}`;
  };

  const [currentDate, setCurrentDate] = useState(getTodayStr());

  // --- Local Input States ---
  const [localResolution, setLocalResolution] = useState('');
  const [localTodos, setLocalTodos] = useState([]);
  const saveTimeoutRef = useRef(null);

  // --- Stopwatch State ---
  const [isTimerRunning, setIsTimerRunning] = useState(false);
  const [seconds, setSeconds] = useState(0);
  const timerRef = useRef(null);

  const emptyRecord = {
    date: currentDate,
    resolution: "",
    todos: [],
    totalTime: 0 
  };

  const currentRecord = records[currentDate] || { ...emptyRecord, date: currentDate };

  // --- Firebase Sync Logic ---
  useEffect(() => {
    let unsubscribeDDay = null;
    let unsubscribeRecords = null;

    const performSignIn = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          try { await signInWithCustomToken(auth, __initial_auth_token); } catch { await signInAnonymously(auth); }
        } else { await signInAnonymously(auth); }
      } catch (err) { setLoading(false); }
    };

    performSignIn();

    const unsubscribeAuth = onAuthStateChanged(auth, (currentUser) => {
      if (currentUser) {
        setUser(currentUser);
        const dDayDocRef = doc(db, 'artifacts', APP_ID, 'users', currentUser.uid, 'settings', 'dday');
        unsubscribeDDay = onSnapshot(dDayDocRef, (docSnap) => {
          if (docSnap.exists()) setDDayTarget(docSnap.data().target || '');
        }, (err) => console.error(err));

        const recordsColRef = collection(db, 'artifacts', APP_ID, 'users', currentUser.uid, 'records');
        unsubscribeRecords = onSnapshot(recordsColRef, (querySnapshot) => {
          const data = {};
          querySnapshot.forEach((doc) => { data[doc.id] = doc.data(); });
          setRecords(data);
          setLoading(false);
        }, () => setLoading(false));
      } else {
        setUser(null);
      }
    });

    return () => {
      if (unsubscribeAuth) unsubscribeAuth();
      if (unsubscribeDDay) unsubscribeDDay();
      if (unsubscribeRecords) unsubscribeRecords();
    };
  }, []);

  useEffect(() => {
    if (currentRecord) {
      setLocalResolution(currentRecord.resolution || '');
      setLocalTodos(currentRecord.todos || []);
      setSeconds(currentRecord.totalTime || 0);
    }
  }, [currentDate, records]);

  const debouncedSave = (updates) => {
    if (saveTimeoutRef.current) clearTimeout(saveTimeoutRef.current);
    saveTimeoutRef.current = setTimeout(() => {
      saveRecordToCloud(updates);
    }, 500);
  };

  const saveRecordToCloud = async (updates) => {
    if (!user) return;
    const recordRef = doc(db, 'artifacts', APP_ID, 'users', user.uid, 'records', currentDate);
    try {
      await setDoc(recordRef, { ...currentRecord, ...updates }, { merge: true });
    } catch (e) { console.error(e); }
  };

  const handleResolutionChange = (val) => {
    setLocalResolution(val);
    debouncedSave({ resolution: val });
  };

  const handleTodoTextChange = (id, text) => {
    const newTodos = localTodos.map(t => t.id === id ? { ...t, text } : t);
    setLocalTodos(newTodos);
    debouncedSave({ todos: newTodos });
  };

  const handleDDayChange = async (val) => {
    setDDayTarget(val);
    if (!user) return;
    const dDayDocRef = doc(db, 'artifacts', APP_ID, 'users', user.uid, 'settings', 'dday');
    await setDoc(dDayDocRef, { target: val });
  };

  const formatDisplayDate = (dateStr) => {
    const year = dateStr.substring(0, 4);
    const month = dateStr.substring(4, 6);
    const day = dateStr.substring(6, 8);
    const d = new Date(`${year}-${month}-${day}`);
    const dayOfWeek = ["일", "월", "화", "수", "목", "금", "토"][d.getDay()];
    return `${year}/${month}/${day}/(${dayOfWeek})`;
  };

  const formatTimeMain = (totalSeconds) => {
    const mins = Math.floor((totalSeconds % 3600) / 60);
    const secs = totalSeconds % 60;
    return `${String(mins).padStart(2, '0')}:${String(secs).padStart(2, '0')}`;
  };

  const formatTimeSub = (totalSeconds) => {
    const hrs = Math.floor(totalSeconds / 3600);
    return `${hrs}H`;
  };

  const toggleTimer = () => {
    if (isTimerRunning) {
      clearInterval(timerRef.current);
      saveRecordToCloud({ totalTime: seconds });
    } else {
      timerRef.current = setInterval(() => { setSeconds(prev => prev + 1); }, 1000);
    }
    setIsTimerRunning(!isTimerRunning);
  };

  const resetTimer = () => {
    if (timerRef.current) clearInterval(timerRef.current);
    setIsTimerRunning(false);
    setSeconds(0);
    saveRecordToCloud({ totalTime: 0 });
  };

  const addTodo = () => {
    const newTodo = { id: Date.now(), text: "", completed: false };
    const newTodos = [...localTodos, newTodo];
    setLocalTodos(newTodos);
    saveRecordToCloud({ todos: newTodos });
  };

  const toggleTodo = (id) => {
    const newTodos = localTodos.map(t => t.id === id ? { ...t, completed: !t.completed } : t);
    setLocalTodos(newTodos);
    saveRecordToCloud({ todos: newTodos });
  };

  const removeTodo = (id) => {
    const newTodos = localTodos.filter(t => t.id !== id);
    setLocalTodos(newTodos);
    saveRecordToCloud({ todos: newTodos });
  };

  const calculateDDay = () => {
    if (!dDayTarget) return "D-?";
    const target = new Date(dDayTarget);
    const today = new Date();
    today.setHours(0,0,0,0);
    const diffDays = Math.ceil((target - today) / (1000 * 60 * 60 * 24));
    return diffDays === 0 ? "D-Day" : diffDays > 0 ? `D-${diffDays}` : `D+${Math.abs(diffDays)}`;
  };

  const changeDate = (offset) => {
    if (isTimerRunning) {
      clearInterval(timerRef.current);
      setIsTimerRunning(false);
      saveRecordToCloud({ totalTime: seconds });
    }
    const d = new Date(currentDate.replace(/(\d{4})(\d{2})(\d{2})/, '$1-$2-$3'));
    d.setDate(d.getDate() + offset);
    const newDateStr = `${d.getFullYear()}${String(d.getMonth() + 1).padStart(2, '0')}${String(d.getDate()).padStart(2, '0')}`;
    setCurrentDate(newDateStr);
    setView('daily');
  };

  // --- Blue Theme Colors (Deep Blue & Sky Blue) ---
  const theme = { 
    bg: '#F0F9FF',      // 매우 연한 하늘색 배경
    primary: '#E0F2FE', // 하늘색 포인트 배경
    secondary: '#BAE6FD', 
    point: '#0369A1',   // 딥블루 (텍스트 및 포인트)
    sidebar: '#082F49', // 아주 어두운 남색 (바인더 및 타이머 배경)
    timerBg: '#082F49',
    accent: '#7DD3FC'   // 밝은 하늘색 (버튼 및 강조)
  };

  const timetable = [
    ["자율", "문권", "연구", "문박", "B"],
    ["E", "A", "D", "C", "영이"],
    ["A", "B", "영이", "스최", "대손"],
    ["스최", "E", "대손", "대손", "E"],
    ["영유", "대손", "문박", "D", "C"],
    ["D", "영유", "B", "A", "창체"],
    ["문권", "C", "창체", "창체", "창체"]
  ];
  const days = ["월", "화", "수", "목", "금"];

  if (loading) return <div className="h-screen bg-blue-50 flex flex-col items-center justify-center font-bold text-blue-900 gap-4"><div className="w-12 h-12 border-4 border-blue-200 border-t-blue-600 rounded-full animate-spin"></div><p>Syncing Blueprints...</p></div>;

  return (
    <div className="h-screen bg-[#F0F9FF] flex items-center justify-center p-0 md:p-4 overflow-hidden relative">
      {/* Background Decor */}
      <Cloud className="absolute top-10 left-10 text-blue-200/40 w-16 h-16 pointer-events-none" />
      <Moon className="absolute bottom-10 right-10 text-blue-100/50 w-20 h-20 pointer-events-none" />
      <Star className="absolute top-1/4 right-20 text-blue-200/30 w-12 h-12 pointer-events-none" />
      <Wind className="absolute bottom-1/4 left-20 text-blue-100/40 w-14 h-14 pointer-events-none" />

      <div className="w-full max-w-[1100px] h-full md:max-h-[850px] bg-white shadow-2xl flex relative md:border-l-[30px] overflow-hidden rounded-none md:rounded-3xl border-white/50" style={{ borderColor: theme.sidebar }}>
        {/* Binder Rings Decor */}
        <div className="hidden md:flex absolute left-0 top-0 bottom-0 w-8 flex-col justify-around py-6 z-10 pointer-events-none">
          {[...Array(14)].map((_, i) => <div key={i} className="w-10 h-3 bg-gradient-to-r from-blue-900/40 to-transparent rounded-full -ml-5 shadow-sm border border-white/5" />)}
        </div>

        <div className="flex-1 flex flex-col md:ml-6 overflow-hidden">
          {/* Header */}
          <div className="px-3 md:px-5 py-3 flex justify-between items-center border-b border-blue-50 shrink-0 bg-white">
            <div className="flex items-center gap-3 md:gap-5 overflow-hidden">
              <div className="flex items-center gap-1 shrink-0">
                <button onClick={() => changeDate(-1)} className="p-1.5 hover:bg-blue-50 rounded-full transition-colors"><ChevronLeft className="w-6 h-6 text-blue-300" /></button>
                <h1 className="text-xl md:text-2xl font-[1000] tracking-tighter text-gray-900 min-w-[160px] md:min-w-[200px] text-center flex items-center justify-center gap-2">
                  <Cloud className="w-5 h-5 text-blue-500" />
                  {formatDisplayDate(currentDate)}
                </h1>
                <button onClick={() => changeDate(1)} className="p-1.5 hover:bg-blue-50 rounded-full transition-colors"><ChevronRight className="w-6 h-6 text-blue-300" /></button>
              </div>
              <div className="flex items-center gap-3 shrink-0">
                <div className="flex items-center gap-2 px-4 py-2 rounded-full border-2 shadow-sm" style={{ backgroundColor: theme.primary, borderColor: theme.accent }}>
                  <div className="text-xl font-black leading-none tracking-tighter shrink-0" style={{ color: theme.point }}>{calculateDDay()}</div>
                  <div className="h-4 w-[1.5px] bg-blue-200 mx-0.5" />
                  <input type="date" value={dDayTarget} onChange={(e) => handleDDayChange(e.target.value)} className="text-[11px] bg-transparent border-none p-0 focus:ring-0 outline-none font-bold text-blue-600 cursor-pointer w-24" />
                </div>
                <div className="flex gap-1.5 p-1 bg-blue-50/50 rounded-full border-2 border-blue-100 shrink-0">
                  <button onClick={() => setView('daily')} className={`px-4 md:px-6 py-1.5 text-[12px] font-black rounded-full transition-all ${view === 'daily' ? 'bg-white shadow-sm text-blue-800' : 'text-blue-300'}`}>PLANNER</button>
                  <button onClick={() => setView('archive')} className={`px-4 md:px-6 py-1.5 text-[12px] font-black rounded-full transition-all ${view === 'archive' ? 'bg-white shadow-sm text-blue-800' : 'text-blue-300'}`}>ARCHIVE</button>
                </div>
              </div>
            </div>
          </div>

          {view === 'daily' ? (
            <div className="flex-1 flex flex-col md:flex-row overflow-hidden">
              {/* Left Column */}
              <div className="flex-[1.8] border-r border-blue-50 p-4 md:p-5 overflow-hidden flex flex-col">
                <div className="mb-4 p-4 rounded-xl border-2 flex items-center gap-3 shrink-0 shadow-inner" style={{ backgroundColor: '#F0F9FF', borderColor: theme.accent }}>
                   <div className="w-2 h-6 rounded-full" style={{ backgroundColor: theme.point }} />
                   <textarea 
                    placeholder="맑은 정신으로 오늘을 위한 다짐..." 
                    value={localResolution} 
                    onChange={(e) => handleResolutionChange(e.target.value)} 
                    className="flex-1 bg-transparent text-[17px] border-none focus:ring-0 resize-none h-6 leading-tight placeholder:text-blue-200 font-bold text-blue-900" 
                  />
                </div>
                <div className="flex-1 overflow-y-auto scrollbar-hide">
                  <div className="flex justify-between items-center mb-4 sticky top-0 bg-white z-10 pb-2">
                    <div className="flex items-center gap-2">
                      <Wind className="w-5 h-5 text-blue-600" />
                      <span className="text-[11px] font-black text-white tracking-widest px-3.5 py-1.5 rounded-full shadow-md uppercase" style={{ backgroundColor: theme.point }}>Tasks</span>
                    </div>
                    <button onClick={addTodo} className="w-10 h-10 text-white rounded-xl flex items-center justify-center shadow-lg hover:rotate-90 active:scale-95 transition-all" style={{ backgroundColor: theme.sidebar }}><Plus className="w-7 h-7" /></button>
                  </div>
                  <div className="space-y-4 px-1 pb-10">
                    {localTodos.map((todo) => (
                      <div key={todo.id} className="flex items-center gap-4 group">
                        <button onClick={() => toggleTodo(todo.id)} className="shrink-0 relative">
                          <div className={`w-8 h-8 rounded-lg border-2 transition-all flex items-center justify-center ${todo.completed ? 'scale-90 shadow-inner' : 'border-blue-100 bg-blue-50/30 shadow-sm'}`} style={todo.completed ? { borderColor: theme.accent, backgroundColor: theme.accent } : {}}>
                            {/* 완료 시 V자 체크 표시 */}
                            {todo.completed && <Check className="w-5 h-5 text-blue-900" strokeWidth={4} />}
                          </div>
                        </button>
                        <input 
                          type="text" 
                          value={todo.text} 
                          onChange={(e) => handleTodoTextChange(todo.id, e.target.value)} 
                          placeholder="할 일을 입력하세요..." 
                          className={`flex-1 bg-transparent border-none focus:ring-0 text-[18px] p-1 font-bold ${todo.completed ? 'text-blue-200 line-through italic' : 'text-gray-800'}`} 
                        />
                        <button onClick={() => removeTodo(todo.id)} className="opacity-0 group-hover:opacity-100 p-1.5 text-gray-200 hover:text-red-400 transition-opacity"><Trash2 className="w-4 h-4" /></button>
                      </div>
                    ))}
                  </div>
                </div>
              </div>
              
              {/* Right Side */}
              <div className="flex-[1] p-4 md:p-5 flex flex-col gap-4 bg-blue-50/20 md:bg-transparent">
                <div className="flex-1 flex flex-col min-h-[250px] md:min-h-0">
                  <div className="flex items-center gap-2 mb-3">
                    <Cloud className="w-4 h-4 text-blue-600" />
                    <h3 className="text-[10px] font-black text-blue-800 tracking-[0.2em] uppercase">Weekly Timetable</h3>
                  </div>
                  <div className="flex-1 bg-white border-2 border-blue-50 rounded-xl overflow-hidden flex flex-col shadow-sm">
                    <div className="grid grid-cols-5 shrink-0" style={{ backgroundColor: theme.sidebar }}>
                      {days.map(day => <div key={day} className="py-1.5 text-center text-[10px] font-black text-blue-100 border-r border-white/10 uppercase last:border-r-0">{day}</div>)}
                    </div>
                    <div className="flex-1 grid grid-rows-7 overflow-hidden">
                      {timetable.map((row, r) => (
                        <div key={r} className="grid grid-cols-5 border-b border-blue-50 last:border-b-0">
                          {row.map((sub, c) => (
                            <div key={c} className="p-0.5 border-r border-blue-50 flex items-center justify-center text-center last:border-r-0 hover:bg-blue-50 transition-colors">
                              <span className="text-blue-900 text-[12px] md:text-[14px] leading-tight line-clamp-2 font-[1000] tracking-tighter drop-shadow-sm">{sub}</span>
                            </div>
                          ))}
                        </div>
                      ))}
                    </div>
                  </div>
                </div>

                <div className="shrink-0">
                  <div className="rounded-2xl p-3 px-4 flex items-center justify-between shadow-xl border border-white/20" style={{ backgroundColor: theme.timerBg }}>
                    <button onClick={toggleTimer} className={`w-14 h-14 rounded-full flex items-center justify-center transition-all ${isTimerRunning ? 'text-blue-900 shadow-[0_0_20px_rgba(125,211,252,0.5)] scale-105' : 'bg-blue-900/40 hover:bg-blue-800'}`} style={isTimerRunning ? { backgroundColor: theme.accent } : { color: theme.accent }}>
                      {isTimerRunning ? <Pause className="w-7 h-7 fill-current" /> : <Play className="w-7 h-7 fill-current ml-0.5" />}
                    </button>
                    <div className="flex flex-col items-end">
                      <span className="text-[10px] font-bold text-blue-300 tracking-widest uppercase mb-[-4px]">Blue Session</span>
                      <div className="flex items-baseline gap-1">
                        <span className="text-3xl md:text-4xl font-normal tabular-nums digital-font" style={{ color: theme.accent }}>{formatTimeMain(seconds)}</span>
                        <span className="text-[12px] font-bold text-blue-300/80 digital-font">{formatTimeSub(seconds)}</span>
                      </div>
                    </div>
                    <button onClick={resetTimer} className="w-12 h-12 bg-white/5 rounded-full flex items-center justify-center text-blue-400 hover:text-white hover:bg-white/10 transition-colors"><RotateCcw className="w-5 h-5" /></button>
                  </div>
                </div>
              </div>
            </div>
          ) : (
            <div className="flex-1 p-6 md:p-8 overflow-y-auto bg-white scrollbar-hide">
              <h2 className="text-2xl font-black text-blue-900 italic mb-8 uppercase tracking-tighter flex items-center gap-3">
                <Cloud className="w-8 h-8" />
                Blue History
              </h2>
              <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6">
                {Object.keys(records).sort((a,b) => b.localeCompare(a)).map(date => (
                  <button key={date} onClick={() => { setCurrentDate(date); setView('daily'); }} className="p-6 rounded-[24px] border-2 border-blue-50 bg-white shadow-md hover:border-blue-300 hover:shadow-xl transition-all text-left group overflow-hidden relative">
                    <Star className="absolute -top-4 -right-4 w-16 h-16 text-blue-50/50 group-hover:scale-110 transition-transform" />
                    <div className="text-xl font-black text-blue-900 tracking-tighter mb-1 group-hover:translate-x-1 transition-transform">{formatDisplayDate(date)}</div>
                    <div className="text-[12px] text-blue-400 truncate mb-4 italic">"{records[date].resolution || '기록된 다짐이 없습니다'}"</div>
                    <div className="flex justify-between items-center pt-3 border-t border-blue-50">
                      <span className="text-[11px] font-black uppercase tracking-widest text-blue-300">Total Time</span>
                      <span className="text-[14px] font-black digital-font text-blue-700">{formatTimeSub(records[date].totalTime || 0)} {formatTimeMain(records[date].totalTime || 0)}</span>
                    </div>
                  </button>
                ))}
              </div>
            </div>
          )}
        </div>
      </div>

      <style dangerouslySetInnerHTML={{ __html: `
        @import url('https://fonts.googleapis.com/css2?family=Gowun+Dodum:wght@400;700&family=Orbitron:wght@400;700&display=swap');
        body { font-family: 'Gowun Dodum', sans-serif; background-color: #F0F9FF; margin: 0; overflow: hidden; }
        .digital-font { font-family: 'Orbitron', sans-serif; }
        .scrollbar-hide::-webkit-scrollbar { display: none; }
        .line-clamp-2 { display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden; }
        textarea:focus, input:focus { outline: none; box-shadow: none; }
        h1 { font-weight: 900 !important; }
        .rounded-3xl { border-radius: 2rem; }
        .rounded-2xl { border-radius: 1.25rem; }
        .rounded-xl { border-radius: 1rem; }
      `}} />
    </div>
  );
};

export default App;
