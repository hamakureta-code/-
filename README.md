import React, { useState, useEffect, useRef, useCallback } from 'react';
import { Play, Pause, Music, Mic2, Clock, Volume2, MoveRight, RotateCcw, Info } from 'lucide-react';

const App = () => {
  const [activeTab, setActiveTab] = useState('metronome');

  return (
    <div className="min-h-screen bg-slate-900 text-slate-100 font-sans selection:bg-teal-500 selection:text-white">
      {/* Header */}
      <header className="bg-slate-800 border-b border-slate-700 p-4 sticky top-0 z-10 shadow-lg">
        <div className="max-w-md mx-auto flex items-center justify-between">
          <div className="flex items-center gap-2">
            <div className="w-8 h-8 bg-gradient-to-br from-teal-400 to-emerald-600 rounded-full flex items-center justify-center text-slate-900 font-bold">
              C
            </div>
            <h1 className="text-xl font-bold bg-clip-text text-transparent bg-gradient-to-r from-teal-200 to-emerald-400">
              ClariMate
            </h1>
          </div>
          <div className="text-xs text-slate-400">For B♭ Clarinet</div>
        </div>
      </header>

      {/* Main Content */}
      <main className="max-w-md mx-auto p-4 pb-24 space-y-6">
        {activeTab === 'metronome' && <Metronome />}
        {activeTab === 'tuner' && <TunerDrone />}
        {activeTab === 'transposer' && <Transposer />}
        {activeTab === 'timer' && <PracticeTimer />}
      </main>

      {/* Navigation Bar */}
      <nav className="fixed bottom-0 left-0 right-0 bg-slate-800 border-t border-slate-700 pb-safe">
        <div className="max-w-md mx-auto flex justify-around p-2">
          <NavButton 
            active={activeTab === 'metronome'} 
            onClick={() => setActiveTab('metronome')} 
            icon={<Clock size={20} />} 
            label="テンポ" 
          />
          <NavButton 
            active={activeTab === 'tuner'} 
            onClick={() => setActiveTab('tuner')} 
            icon={<Volume2 size={20} />} 
            label="音程" 
          />
          <NavButton 
            active={activeTab === 'transposer'} 
            onClick={() => setActiveTab('transposer')} 
            icon={<Music size={20} />} 
            label="移調" 
          />
          <NavButton 
            active={activeTab === 'timer'} 
            onClick={() => setActiveTab('timer')} 
            icon={<RotateCcw size={20} />} 
            label="記録" 
          />
        </div>
      </nav>
    </div>
  );
};

/* --- Sub Components --- */

const NavButton = ({ active, onClick, icon, label }) => (
  <button
    onClick={onClick}
    className={`flex flex-col items-center justify-center p-2 rounded-lg w-16 transition-all duration-200 ${
      active 
        ? 'text-teal-400 bg-slate-700/50' 
        : 'text-slate-400 hover:text-slate-200 hover:bg-slate-700/30'
    }`}
  >
    <div className="mb-1">{icon}</div>
    <span className="text-[10px] font-medium">{label}</span>
  </button>
);

// --- Metronome Component ---
const Metronome = () => {
  const [isPlaying, setIsPlaying] = useState(false);
  const [bpm, setBpm] = useState(100);
  const [audioContext, setAudioContext] = useState(null);
  const timerRef = useRef(null);
  const nextNoteTimeRef = useRef(0);

  // Initialize AudioContext only on user interaction to comply with browser policies
  const initAudio = () => {
    if (!audioContext) {
      const ctx = new (window.AudioContext || window.webkitAudioContext)();
      setAudioContext(ctx);
      return ctx;
    }
    return audioContext;
  };

  const playClick = (ctx, time) => {
    const osc = ctx.createOscillator();
    const gainNode = ctx.createGain();

    osc.connect(gainNode);
    gainNode.connect(ctx.destination);

    osc.frequency.value = 1200; // High pitch click
    gainNode.gain.setValueAtTime(1, time);
    gainNode.gain.exponentialRampToValueAtTime(0.001, time + 0.1);

    osc.start(time);
    osc.stop(time + 0.1);
  };

  const scheduler = useCallback(() => {
    const lookahead = 25.0; // How far ahead to schedule audio (ms)
    const scheduleAheadTime = 0.1; // How far ahead to schedule audio (sec)

    if (!audioContext) return;

    while (nextNoteTimeRef.current < audioContext.currentTime + scheduleAheadTime) {
      playClick(audioContext, nextNoteTimeRef.current);
      const secondsPerBeat = 60.0 / bpm;
      nextNoteTimeRef.current += secondsPerBeat;
    }
    
    timerRef.current = setTimeout(scheduler, lookahead);
  }, [bpm, audioContext]);

  useEffect(() => {
    if (isPlaying && audioContext) {
      if (audioContext.state === 'suspended') {
        audioContext.resume();
      }
      nextNoteTimeRef.current = audioContext.currentTime + 0.05;
      scheduler();
    } else {
      clearTimeout(timerRef.current);
    }
    return () => clearTimeout(timerRef.current);
  }, [isPlaying, bpm, audioContext, scheduler]);

  const togglePlay = () => {
    const ctx = initAudio();
    if (!isPlaying) {
      // Start
      setIsPlaying(true);
    } else {
      setIsPlaying(false);
    }
  };

  return (
    <div className="bg-slate-800 rounded-2xl p-6 shadow-xl border border-slate-700 flex flex-col items-center space-y-8">
      <div className="text-center space-y-2">
        <h2 className="text-sm font-semibold text-teal-400 tracking-wider uppercase">Metronome</h2>
        <div className="text-6xl font-black tabular-nums text-white tracking-tight">
          {bpm}
        </div>
        <p className="text-slate-400 text-sm">BPM (Tempo)</p>
      </div>

      <div className="w-full space-y-6">
        <input
          type="range"
          min="40"
          max="208"
          value={bpm}
          onChange={(e) => setBpm(Number(e.target.value))}
          className="w-full h-3 bg-slate-700 rounded-lg appearance-none cursor-pointer accent-teal-500 hover:accent-teal-400 transition-all"
        />
        
        <div className="flex justify-between gap-4">
          <button 
            onClick={() => setBpm(prev => Math.max(40, prev - 5))}
            className="flex-1 py-3 bg-slate-700 rounded-lg text-slate-300 font-bold hover:bg-slate-600 active:bg-slate-500 transition-colors"
          >
            -5
          </button>
          <button 
            onClick={() => setBpm(prev => Math.max(40, prev - 1))}
            className="flex-1 py-3 bg-slate-700 rounded-lg text-slate-300 font-bold hover:bg-slate-600 active:bg-slate-500 transition-colors"
          >
            -1
          </button>
          <button 
            onClick={() => setBpm(prev => Math.min(208, prev + 1))}
            className="flex-1 py-3 bg-slate-700 rounded-lg text-slate-300 font-bold hover:bg-slate-600 active:bg-slate-500 transition-colors"
          >
            +1
          </button>
          <button 
            onClick={() => setBpm(prev => Math.min(208, prev + 5))}
            className="flex-1 py-3 bg-slate-700 rounded-lg text-slate-300 font-bold hover:bg-slate-600 active:bg-slate-500 transition-colors"
          >
            +5
          </button>
        </div>
      </div>

      <button
        onClick={togglePlay}
        className={`w-24 h-24 rounded-full flex items-center justify-center shadow-lg transition-all duration-300 ${
          isPlaying 
            ? 'bg-rose-500 text-white shadow-rose-900/50 hover:bg-rose-600 scale-95' 
            : 'bg-teal-500 text-slate-900 shadow-teal-900/50 hover:bg-teal-400 hover:scale-105'
        }`}
      >
        {isPlaying ? <Pause size={40} fill="currentColor" /> : <Play size={40} fill="currentColor" className="ml-1" />}
      </button>
    </div>
  );
};

// --- Tuner/Drone Component ---
const TunerDrone = () => {
  const [isPlaying, setIsPlaying] = useState(false);
  const [note, setNote] = useState('Bb'); // Concert Bb = Clarinet C
  const [audioContext, setAudioContext] = useState(null);
  const oscRef = useRef(null);
  const gainRef = useRef(null);

  // Frequencies for Concert Pitch (Clarinet plays 2 semitones higher)
  // Clarinet C = Concert Bb (466.16 Hz)
  // Clarinet B = Concert A (440.00 Hz)
  const frequencies = {
    'Bb': { freq: 466.16, label: '実音 B♭ (クラリネット C)' },
    'A': { freq: 440.00, label: '実音 A (クラリネット B)' },
  };

  const stopSound = () => {
    if (oscRef.current) {
      try {
        const now = audioContext.currentTime;
        gainRef.current.gain.exponentialRampToValueAtTime(0.001, now + 0.1);
        oscRef.current.stop(now + 0.1);
      } catch (e) { /* ignore */ }
      oscRef.current = null;
    }
    setIsPlaying(false);
  };

  const playSound = () => {
    if (isPlaying) {
      stopSound();
      return;
    }

    let ctx = audioContext;
    if (!ctx) {
      ctx = new (window.AudioContext || window.webkitAudioContext)();
      setAudioContext(ctx);
    }
    if (ctx.state === 'suspended') ctx.resume();

    const osc = ctx.createOscillator();
    const gain = ctx.createGain();
    
    // Use triangle wave for a softer, woodwind-like sound
    osc.type = 'triangle';
    osc.frequency.value = frequencies[note].freq;

    osc.connect(gain);
    gain.connect(ctx.destination);

    const now = ctx.currentTime;
    gain.gain.setValueAtTime(0, now);
    gain.gain.linearRampToValueAtTime(0.2, now + 0.1); // Lower volume

    osc.start(now);
    
    oscRef.current = osc;
    gainRef.current = gain;
    setIsPlaying(true);
  };

  // Change frequency while playing if note changes
  useEffect(() => {
    if (isPlaying && oscRef.current) {
      const now = audioContext.currentTime;
      oscRef.current.frequency.setValueAtTime(frequencies[note].freq, now);
    }
  }, [note]);

  // Cleanup
  useEffect(() => {
    return () => stopSound();
  }, []);

  return (
    <div className="space-y-6">
      <div className="bg-slate-800 rounded-2xl p-6 shadow-xl border border-slate-700">
        <h2 className="text-center text-sm font-semibold text-teal-400 mb-6 uppercase tracking-wider">Tuning Drone</h2>
        
        <div className="flex flex-col gap-4 mb-8">
          <label className={`relative flex items-center p-4 rounded-xl border-2 cursor-pointer transition-all ${note === 'Bb' ? 'border-teal-500 bg-teal-500/10' : 'border-slate-600 hover:border-slate-500'}`}>
            <input 
              type="radio" 
              name="tuningNote" 
              value="Bb" 
              checked={note === 'Bb'} 
              onChange={() => setNote('Bb')}
              className="absolute opacity-0"
            />
            <div className={`w-6 h-6 rounded-full border-2 mr-4 flex items-center justify-center ${note === 'Bb' ? 'border-teal-500' : 'border-slate-400'}`}>
              {note === 'Bb' && <div className="w-3 h-3 rounded-full bg-teal-500" />}
            </div>
            <div>
              <div className="font-bold text-lg">Concert B♭ (実音)</div>
              <div className="text-slate-400 text-sm">クラリネットの「ド (C)」</div>
            </div>
          </label>

          <label className={`relative flex items-center p-4 rounded-xl border-2 cursor-pointer transition-all ${note === 'A' ? 'border-teal-500 bg-teal-500/10' : 'border-slate-600 hover:border-slate-500'}`}>
            <input 
              type="radio" 
              name="tuningNote" 
              value="A" 
              checked={note === 'A'} 
              onChange={() => setNote('A')}
              className="absolute opacity-0"
            />
            <div className={`w-6 h-6 rounded-full border-2 mr-4 flex items-center justify-center ${note === 'A' ? 'border-teal-500' : 'border-slate-400'}`}>
              {note === 'A' && <div className="w-3 h-3 rounded-full bg-teal-500" />}
            </div>
            <div>
              <div className="font-bold text-lg">Concert A (実音)</div>
              <div className="text-slate-400 text-sm">クラリネットの「シ (B)」</div>
            </div>
          </label>
        </div>

        <div className="flex justify-center">
          <button
            onClick={playSound}
            className={`flex items-center gap-3 px-8 py-4 rounded-full font-bold text-lg shadow-lg transition-all ${
              isPlaying 
                ? 'bg-rose-500 text-white hover:bg-rose-600' 
                : 'bg-teal-500 text-slate-900 hover:bg-teal-400'
            }`}
          >
            {isPlaying ? (
              <>
                <Volume2 className="animate-pulse" /> 停止 (Stop)
              </>
            ) : (
              <>
                <Play /> 再生 (Play)
              </>
            )}
          </button>
        </div>
      </div>
      
      <div className="bg-slate-800/50 p-4 rounded-xl border border-slate-700/50">
        <div className="flex items-start gap-3">
          <Info className="text-teal-400 shrink-0 mt-1" size={20} />
          <p className="text-sm text-slate-300 leading-relaxed">
            ロングトーン練習の際に、この音に合わせて音程が揺れないように吹く練習が効果的です。クラリネットの「ド」は実音B♭です。
          </p>
        </div>
      </div>
    </div>
  );
};

// --- Transposer Component ---
const Transposer = () => {
  const notes = ['C', 'C#', 'D', 'Eb', 'E', 'F', 'F#', 'G', 'Ab', 'A', 'Bb', 'B'];
  const [inputNote, setInputNote] = useState('C');
  
  // Convert Concert Pitch (Piano) to Clarinet (Bb)
  // Clarinet sounds 2 semitones LOWER than written.
  // So if Piano plays C, Clarinet must play D to sound like C.
  // Formula: Concert Pitch + 2 semitones = Clarinet Pitch
  
  const getNoteIndex = (n) => notes.indexOf(n);
  
  const calculateClarinetNote = (concertNote) => {
    const idx = getNoteIndex(concertNote);
    // +2 semitones
    const newIdx = (idx + 2) % 12;
    return notes[newIdx];
  };

  const clarinetNote = calculateClarinetNote(inputNote);

  return (
    <div className="bg-slate-800 rounded-2xl p-6 shadow-xl border border-slate-700 space-y-8">
      <h2 className="text-center text-sm font-semibold text-teal-400 uppercase tracking-wider">
        移調計算機 (実音 → B♭管)
      </h2>

      <div className="grid grid-cols-1 md:grid-cols-3 gap-6 items-center text-center">
        {/* Input (Concert) */}
        <div className="space-y-2">
          <label className="text-xs text-slate-400 font-bold uppercase block">ピアノ・フルートの楽譜</label>
          <div className="relative">
            <select 
              value={inputNote} 
              onChange={(e) => setInputNote(e.target.value)}
              className="w-full bg-slate-700 text-white text-2xl font-bold py-4 px-8 rounded-xl appearance-none outline-none focus:ring-2 focus:ring-teal-500 text-center cursor-pointer"
            >
              {notes.map(n => <option key={n} value={n}>{n}</option>)}
            </select>
            <div className="absolute right-4 top-1/2 -translate-y-1/2 pointer-events-none text-slate-400">
              ▼
            </div>
          </div>
          <div className="text-sm text-slate-500">実音 (Concert Pitch)</div>
        </div>

        {/* Arrow */}
        <div className="flex justify-center text-teal-500">
          <MoveRight size={32} className="md:rotate-0 rotate-90" />
        </div>

        {/* Output (Clarinet) */}
        <div className="space-y-2">
          <label className="text-xs text-teal-400 font-bold uppercase block">クラリネットで吹く音</label>
          <div className="bg-gradient-to-br from-teal-500 to-emerald-600 text-slate-900 text-4xl font-black py-4 rounded-xl shadow-lg shadow-teal-900/50 flex items-center justify-center">
            {clarinetNote}
          </div>
          <div className="text-sm text-slate-500">in B♭</div>
        </div>
      </div>

      <div className="border-t border-slate-700 pt-6">
        <h3 className="text-slate-300 font-bold mb-2 text-sm">簡単な読み替え表</h3>
        <div className="grid grid-cols-4 gap-2 text-xs text-center">
          <div className="bg-slate-700/50 p-2 rounded">C → D</div>
          <div className="bg-slate-700/50 p-2 rounded">F → G</div>
          <div className="bg-slate-700/50 p-2 rounded">Bb → C</div>
          <div className="bg-slate-700/50 p-2 rounded">Eb → F</div>
        </div>
      </div>
    </div>
  );
};

// --- Practice Timer Component ---
const PracticeTimer = () => {
  const [time, setTime] = useState(0);
  const [isRunning, setIsRunning] = useState(false);
  const timerIntervalRef = useRef(null);

  useEffect(() => {
    if (isRunning) {
      timerIntervalRef.current = setInterval(() => {
        setTime(prev => prev + 1);
      }, 1000);
    } else {
      clearInterval(timerIntervalRef.current);
    }
    return () => clearInterval(timerIntervalRef.current);
  }, [isRunning]);

  const formatTime = (seconds) => {
    const h = Math.floor(seconds / 3600);
    const m = Math.floor((seconds % 3600) / 60);
    const s = seconds % 60;
    return `${h > 0 ? h + ':' : ''}${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
  };

  const resetTimer = () => {
    setIsRunning(false);
    setTime(0);
  };

  return (
    <div className="bg-slate-800 rounded-2xl p-6 shadow-xl border border-slate-700 text-center space-y-8">
      <div className="space-y-4">
        <h2 className="text-sm font-semibold text-teal-400 tracking-wider uppercase">Session Timer</h2>
        <div className="text-7xl font-mono text-white tracking-tighter tabular-nums">
          {formatTime(time)}
        </div>
        <p className="text-slate-400">今日の練習時間</p>
      </div>

      <div className="grid grid-cols-2 gap-4">
        <button
          onClick={() => setIsRunning(!isRunning)}
          className={`py-4 rounded-xl font-bold text-lg transition-all ${
            isRunning 
              ? 'bg-rose-500/10 text-rose-500 border-2 border-rose-500 hover:bg-rose-500 hover:text-white' 
              : 'bg-teal-500 text-slate-900 border-2 border-teal-500 hover:bg-teal-400 hover:border-teal-400'
          }`}
        >
          {isRunning ? '一時停止' : 'スタート'}
        </button>
        <button
          onClick={resetTimer}
          className="py-4 rounded-xl font-bold text-lg bg-slate-700 text-slate-300 hover:bg-slate-600 transition-all border-2 border-transparent"
        >
          リセット
        </button>
      </div>

      <div className="text-xs text-slate-500 text-left bg-slate-900/50 p-4 rounded-lg">
        <p className="mb-2 font-bold text-slate-400">練習のヒント:</p>
        <ul className="list-disc pl-4 space-y-1">
          <li>最初の5分はロングトーンに使いましょう。</li>
          <li>難しいパッセージはテンポを落として練習しましょう。</li>
          <li>25分練習したら、5分休憩しましょう（ポモドーロ法）。</li>
        </ul>
      </div>
    </div>
  );
};

export default App;
