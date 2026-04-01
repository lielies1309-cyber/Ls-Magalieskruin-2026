# Ls-Maimport React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getFirestore, 
  collection, 
  addDoc, 
  onSnapshot, 
  query, 
  doc, 
  deleteDoc, 
  updateDoc 
} from 'firebase/firestore';
import { 
  getAuth, 
  signInAnonymously, 
  signInWithCustomToken, 
  onAuthStateChanged 
} from 'firebase/auth';
import { Trophy, MapPin, Calendar, Users, Plus, Trash2, Info, Star, UserPlus, X, Edit3, History, Target } from 'lucide-react';

// Firebase configuration from environment
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'ls-magalieskruin-2026';

export default function App() {
  const [user, setUser] = useState(null);
  const [matches, setMatches] = useState([]);
  const [loading, setLoading] = useState(true);
  const [showForm, setShowForm] = useState(false);
  const [editingId, setEditingId] = useState(null);
  const formRef = useRef(null);

  // Form State
  const [formData, setFormData] = useState({
    date: new Date().toISOString().split('T')[0],
    opponent: '',
    location: '',
    ourScore: '',
    theirScore: '',
    scorers: []
  });

  const [currentScorer, setCurrentScorer] = useState({ name: '', tries: 0, conversions: 0, penalties: 0 });

  // 1. Handle Authentication
  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (error) {
        console.error("Auth error:", error);
      }
    };
    initAuth();

    const unsubscribe = onAuthStateChanged(auth, (user) => {
      setUser(user);
    });
    return () => unsubscribe();
  }, []);

  // 2. Listen to Firestore Data
  useEffect(() => {
    if (!user) return;

    const matchesRef = collection(db, 'artifacts', appId, 'public', 'data', 'matches');
    
    const unsubscribe = onSnapshot(matchesRef, 
      (snapshot) => {
        const matchData = snapshot.docs.map(doc => ({
          id: doc.id,
          ...doc.data()
        }));
        
        matchData.sort((a, b) => new Date(b.date) - new Date(a.date));
        setMatches(matchData);
        setLoading(false);
      },
      (error) => {
        console.error("Firestore error:", error);
        setLoading(false);
      }
    );

    return () => unsubscribe();
  }, [user]);

  const addScorerToForm = () => {
    if (!currentScorer.name) return;
    setFormData({
      ...formData,
      scorers: [...formData.scorers, { ...currentScorer }]
    });
    setCurrentScorer({ name: '', tries: 0, conversions: 0, penalties: 0 });
  };

  const removeScorerFromForm = (index) => {
    const newScorers = [...formData.scorers];
    newScorers.splice(index, 1);
    setFormData({ ...formData, scorers: newScorers });
  };

  const handleSaveMatch = async (e) => {
    e.preventDefault();
    if (!user) return;

    const ourScoreNum = parseInt(formData.ourScore) || 0;
    const theirScoreNum = parseInt(formData.theirScore) || 0;
    
    let result = 'Draw';
    if (ourScoreNum > theirScoreNum) result = 'Win';
    else if (ourScoreNum < theirScoreNum) result = 'Loss';

    const finalData = {
      ...formData,
      ourScore: ourScoreNum,
      theirScore: theirScoreNum,
      result,
      updatedAt: new Date().toISOString()
    };

    try {
      if (editingId) {
        const matchDoc = doc(db, 'artifacts', appId, 'public', 'data', 'matches', editingId);
        await updateDoc(matchDoc, finalData);
      } else {
        const matchesRef = collection(db, 'artifacts', appId, 'public', 'data', 'matches');
        await addDoc(matchesRef, {
          ...finalData,
          addedBy: user.uid,
          createdAt: new Date().toISOString()
        });
      }

      resetForm();
    } catch (error) {
      console.error("Error saving match:", error);
    }
  };

  const resetForm = () => {
    setFormData({
      date: new Date().toISOString().split('T')[0],
      opponent: '',
      location: '',
      ourScore: '',
      theirScore: '',
      scorers: []
    });
    setEditingId(null);
    setShowForm(false);
  };

  const startEdit = (match) => {
    setFormData({
      date: match.date,
      opponent: match.opponent,
      location: match.location,
      ourScore: match.ourScore.toString(),
      theirScore: match.theirScore.toString(),
      scorers: match.scorers || []
    });
    setEditingId(match.id);
    setShowForm(true);
    window.scrollTo({ top: 0, behavior: 'smooth' });
  };

  const deleteMatch = async (id) => {
    try {
      const matchDoc = doc(db, 'artifacts', appId, 'public', 'data', 'matches', id);
      await deleteDoc(matchDoc);
    } catch (error) {
      console.error("Error deleting match:", error);
    }
  };

  // Leaderboard Calculation
  const leaderboard = matches.reduce((acc, match) => {
    (match.scorers || []).forEach(s => {
      const points = (parseInt(s.tries || 0) * 5) + 
                     (parseInt(s.conversions || 0) * 2) + 
                     (parseInt(s.penalties || 0) * 3);
      if (!acc[s.name]) {
        acc[s.name] = { name: s.name, tries: 0, conversions: 0, penalties: 0, points: 0 };
      }
      acc[s.name].tries += parseInt(s.tries || 0);
      acc[s.name].conversions += parseInt(s.conversions || 0);
      acc[s.name].penalties += parseInt(s.penalties || 0);
      acc[s.name].points += points;
    });
    return acc;
  }, {});

  const sortedLeaderboard = Object.values(leaderboard).sort((a, b) => b.points - a.points);

  const stats = matches.reduce((acc, match) => {
    acc.played += 1;
    if (match.result === 'Win') acc.wins += 1;
    else if (match.result === 'Loss') acc.losses += 1;
    else acc.draws += 1;
    return acc;
  }, { played: 0, wins: 0, losses: 0, draws: 0 });

  if (loading) {
    return (
      <div className="min-h-screen bg-slate-50 flex items-center justify-center p-4">
        <div className="text-center">
          <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-green-700 mx-auto"></div>
          <p className="mt-4 text-slate-600 font-medium text-lg">Accessing Ls Magalieskruin Records...</p>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-slate-50 text-slate-900 pb-24 font-sans">
      {/* Header */}
      <header className="bg-green-800 text-white shadow-lg p-6 sticky top-0 z-30">
        <div className="max-w-3xl mx-auto flex justify-between items-center">
          <div>
            <h1 className="text-xl md:text-2xl font-black flex items-center gap-2 uppercase tracking-tight">
              <Trophy className="text-yellow-400 shrink-0" />
              Ls Magalieskruin 2026
            </h1>
            <p className="text-green-100 text-[10px] font-bold uppercase tracking-widest opacity-80">Rugby Team Manager</p>
          </div>
          <button 
            onClick={() => { if(showForm) resetForm(); else setShowForm(true); }}
            className={`${showForm ? 'bg-red-500 hover:bg-red-600' : 'bg-white hover:bg-green-50'} p-2 rounded-full transition-all shadow-md ml-2`}
          >
            {showForm ? <X className="text-white" /> : <Plus className="text-green-800" />}
          </button>
        </div>
      </header>

      <main className="max-w-3xl mx-auto p-4 space-y-8">
        {/* Statistics & Leaderboard Overview */}
        <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
          <section className="md:col-span-1 grid grid-cols-2 gap-2 text-center h-fit">
            <div className="bg-white p-3 rounded-xl shadow-sm border border-slate-200">
              <div className="text-[9px] text-slate-400 uppercase font-black tracking-widest">Played</div>
              <div className="text-xl font-black text-slate-800">{stats.played}</div>
            </div>
            <div className="bg-green-50 p-3 rounded-xl shadow-sm border border-green-100">
              <div className="text-[9px] text-green-600 uppercase font-black tracking-widest">Wins</div>
              <div className="text-xl font-black text-green-700">{stats.wins}</div>
            </div>
            <div className="bg-slate-50 p-3 rounded-xl shadow-sm border border-slate-200">
              <div className="text-[9px] text-slate-400 uppercase font-black tracking-widest">Draws</div>
              <div className="text-xl font-black text-slate-700">{stats.draws}</div>
            </div>
            <div className="bg-red-50 p-3 rounded-xl shadow-sm border border-red-100">
              <div className="text-[9px] text-red-600 uppercase font-black tracking-widest">Losses</div>
              <div className="text-xl font-black text-red-800">{stats.losses}</div>
            </div>
          </section>

          {/* Points Leaderboard */}
          <section className="md:col-span-2 bg-white rounded-2xl shadow-sm border border-slate-200 overflow-hidden">
            <div className="bg-slate-800 text-white px-4 py-2.5 flex items-center gap-2 font-bold text-[10px] uppercase tracking-[0.2em]">
              <Star className="h-3 w-3 text-yellow-400" /> Scoring Leaders
            </div>
            <div className="max-h-40 overflow-y-auto">
              <table className="w-full text-left text-sm">
                <thead className="bg-slate-50 text-[9px] text-slate-500 uppercase font-bold sticky top-0">
                  <tr>
                    <th className="px-4 py-2">Player</th>
                    <th className="px-2 py-2 text-center">T</th>
                    <th className="px-2 py-2 text-center">C</th>
                    <th className="px-2 py-2 text-center">P</th>
                    <th className="px-4 py-2 text-right">Points</th>
                  </tr>
                </thead>
                <tbody className="divide-y divide-slate-100">
                  {sortedLeaderboard.length === 0 ? (
                    <tr><td colSpan="5" className="px-4 py-8 text-center text-slate-400 italic">No points recorded yet</td></tr>
                  ) : (
                    sortedLeaderboard.map((player, idx) => (
                      <tr key={idx} className={idx === 0 ? "bg-yellow-50/40" : "hover:bg-slate-50"}>
                        <td className="px-4 py-2.5 font-bold text-slate-700">{player.name}</td>
                        <td className="px-2 py-2.5 text-center text-slate-500 font-medium">{player.tries}</td>
                        <td className="px-2 py-2.5 text-center text-slate-500 font-medium">{player.conversions}</td>
                        <td className="px-2 py-2.5 text-center text-slate-500 font-medium">{player.penalties}</td>
                        <td className="px-4 py-2.5 text-right font-black text-green-700">{player.points} <span className="text-[9px] font-normal text-slate-400">pts</span></td>
                      </tr>
                    ))
                  )}
                </tbody>
              </table>
            </div>
          </section>
        </div>

        {/* Add/Edit Match Form */}
        {showForm && (
          <form 
            ref={formRef}
            onSubmit={handleSaveMatch} 
            className="bg-white p-6 rounded-2xl shadow-2xl border-2 border-green-100 space-y-4 animate-in fade-in slide-in-from-top-4 duration-300 relative overflow-hidden"
          >
            {editingId && <div className="absolute top-0 left-0 right-0 bg-yellow-400 h-1" />}
            <div className="flex justify-between items-center border-b pb-3">
              <h2 className="font-black text-lg text-green-800 uppercase tracking-tight flex items-center gap-2">
                {editingId ? <Edit3 className="h-5 w-5" /> : <Plus className="h-5 w-5" />}
                {editingId ? 'Edit Match Record' : 'Log New 2026 Match'}
              </h2>
              {editingId && (
                <button type="button" onClick={resetForm} className="text-[10px] bg-slate-100 hover:bg-slate-200 px-2 py-1 rounded font-bold text-slate-500 uppercase">
                  Cancel Edit
                </button>
              )}
            </div>
            
            <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
              <div className="sm:col-span-2">
                <label className="block text-[10px] font-black text-slate-400 uppercase tracking-widest mb-1">Opponent School</label>
                <input required type="text" placeholder="e.g. Laerskool Rooihuiskraal" className="w-full p-3 bg-slate-50 border border-slate-200 rounded-xl outline-none focus:ring-2 focus:ring-green-500 transition-all font-bold"
                  value={formData.opponent} onChange={e => setFormData({...formData, opponent: e.target.value})} />
              </div>
              <div>
                <label className="block text-[10px] font-black text-slate-400 uppercase tracking-widest mb-1">Match Date</label>
                <input required type="date" className="w-full p-3 bg-slate-50 border border-slate-200 rounded-xl font-medium"
                  value={formData.date} onChange={e => setFormData({...formData, date: e.target.value})} />
              </div>
              <div>
                <label className="block text-[10px] font-black text-slate-400 uppercase tracking-widest mb-1">Venue / City</label>
                <input required type="text" placeholder="e.g. Pretoria North" className="w-full p-3 bg-slate-50 border border-slate-200 rounded-xl font-medium"
                  value={formData.location} onChange={e => setFormData({...formData, location: e.target.value})} />
              </div>
            </div>

            {/* Scorers Section */}
            <div className="bg-slate-50 p-4 rounded-2xl space-y-3 border border-slate-200">
              <h3 className="text-[10px] font-black text-slate-600 uppercase flex items-center gap-2 tracking-[0.1em]">
                <Users className="h-3 w-3" /> Player Stats (T=5p, C=2p, P=3p)
              </h3>
              
              <div className="flex gap-2 flex-wrap md:flex-nowrap">
                <input type="text" placeholder="Player Name" className="flex-1 min-w-[120px] p-2.5 text-sm bg-white border border-slate-200 rounded-lg outline-none focus:ring-2 focus:ring-green-500" 
                  value={currentScorer.name} onChange={e => setCurrentScorer({...currentScorer, name: e.target.value})} />
                <div className="flex gap-2">
                  <div className="flex flex-col items-center">
                    <span className="text-[8px] font-black text-slate-400 mb-1">T</span>
                    <input type="number" className="w-12 p-2 text-sm bg-white border border-slate-200 rounded-lg text-center font-bold" 
                      value={currentScorer.tries} onChange={e => setCurrentScorer({...currentScorer, tries: parseInt(e.target.value) || 0})} />
                  </div>
                  <div className="flex flex-col items-center">
                    <span className="text-[8px] font-black text-slate-400 mb-1">C</span>
                    <input type="number" className="w-12 p-2 text-sm bg-white border border-slate-200 rounded-lg text-center font-bold" 
                      value={currentScorer.conversions} onChange={e => setCurrentScorer({...currentScorer, conversions: parseInt(e.target.value) || 0})} />
                  </div>
                  <div className="flex flex-col items-center">
                    <span className="text-[8px] font-black text-slate-400 mb-1">P</span>
                    <input type="number" className="w-12 p-2 text-sm bg-white border border-slate-200 rounded-lg text-center font-bold" 
                      value={currentScorer.penalties} onChange={e => setCurrentScorer({...currentScorer, penalties: parseInt(e.target.value) || 0})} />
                  </div>
                </div>
                <button type="button" onClick={addScorerToForm} className="bg-green-600 self-end text-white p-2.5 rounded-lg hover:bg-green-700 transition-colors">
                  <UserPlus className="h-4 w-4" />
                </button>
              </div>

              <div className="grid grid-cols-1 sm:grid-cols-2 gap-2 max-h-40 overflow-y-auto pr-1">
                {formData.scorers.map((s, idx) => (
                  <div key={idx} className="flex justify-between items-center bg-white p-3 rounded-xl border border-slate-100 text-sm shadow-sm">
                    <span className="font-bold text-slate-700">{s.name}</span>
                    <div className="flex items-center gap-3">
                      <span className="text-[10px] font-black text-slate-400 bg-slate-50 px-2 py-1 rounded">
                        {s.tries}T • {s.conversions}C • {s.penalties || 0}P
                      </span>
                      <button type="button" onClick={() => removeScorerFromForm(idx)} className="text-red-300 hover:text-red-500">
                        <X className="h-4 w-4" />
                      </button>
                    </div>
                  </div>
                ))}
              </div>
            </div>

            <div className="grid grid-cols-2 gap-4 pt-2">
              <div>
                <label className="block text-[10px] font-black text-green-700 uppercase tracking-widest mb-1 text-center">Magalies Score</label>
                <input required type="number" className="w-full p-4 bg-green-50 border-2 border-green-200 rounded-2xl text-center text-3xl font-black text-green-800 outline-none focus:border-green-500"
                  value={formData.ourScore} onChange={e => setFormData({...formData, ourScore: e.target.value})} />
              </div>
              <div>
                <label className="block text-[10px] font-black text-slate-400 uppercase tracking-widest mb-1 text-center">Opponent Score</label>
                <input required type="number" className="w-full p-4 bg-slate-50 border-2 border-slate-200 rounded-2xl text-center text-3xl font-black text-slate-800 outline-none focus:border-slate-500"
                  value={formData.theirScore} onChange={e => setFormData({...formData, theirScore: e.target.value})} />
              </div>
            </div>

            <button type="submit" className="w-full bg-green-700 text-white font-black py-4 rounded-2xl hover:bg-green-800 shadow-xl active:scale-[0.98] transition-all uppercase tracking-[0.15em] text-sm mt-2">
              {editingId ? 'Update Match History' : 'Save Final Score'}
            </button>
          </form>
        )}

        {/* History Section */}
        <section className="space-y-4">
          <div className="flex items-center gap-2 text-slate-400 font-black text-xs uppercase tracking-[0.2em] px-2">
            <History className="h-4 w-4" /> Season History
          </div>

          {matches.map((match) => (
            <div key={match.id} className="bg-white rounded-3xl shadow-md border border-slate-200 overflow-hidden relative group hover:border-green-200 transition-colors">
              <div className={`absolute top-0 right-0 px-5 py-1.5 text-[9px] font-black uppercase tracking-[0.2em] rounded-bl-2xl ${
                match.result === 'Win' ? 'bg-green-600 text-white' :
                match.result === 'Loss' ? 'bg-red-600 text-white' :
                'bg-slate-400 text-white'
              }`}>
                {match.result}
              </div>

              <div className="p-6 flex flex-col md:flex-row md:items-center justify-between gap-6">
                <div className="flex-1 space-y-2">
                  <div className="flex items-center gap-2 text-slate-400 text-[10px] font-black uppercase tracking-widest">
                    <Calendar className="h-3 w-3" />
                    {new Date(match.date).toLocaleDateString(undefined, { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' })}
                  </div>
                  
                  <h3 className="text-2xl font-black text-slate-800 tracking-tight leading-none">
                    vs {match.opponent}
                  </h3>

                  <div className="flex items-center gap-2 text-slate-400 text-xs font-bold">
                    <MapPin className="h-3 w-3" />
                    {match.location}
                  </div>
                </div>

                <div className="flex items-center gap-4 bg-slate-50 px-6 py-4 rounded-[2rem] border border-slate-100 min-w-[160px] justify-center shadow-inner">
                  <div className="text-center">
                    <div className="text-[9px] font-black text-slate-400 uppercase tracking-widest mb-1">LS Mag</div>
                    <div className={`text-3xl font-black ${match.result === 'Win' ? 'text-green-700' : 'text-slate-800'}`}>{match.ourScore}</div>
                  </div>
                  <div className="text-slate-200 font-light text-2xl mb-[-4px]">|</div>
                  <div className="text-center">
                    <div className="text-[9px] font-black text-slate-400 uppercase tracking-widest mb-1">Opp</div>
                    <div className={`text-3xl font-black ${match.result === 'Loss' ? 'text-red-700' : 'text-slate-800'}`}>{match.theirScore}</div>
                  </div>
                </div>
              </div>

              {/* Match Scorers List */}
              {match.scorers && match.scorers.length > 0 && (
                <div className="px-6 pb-6 pt-0">
                  <div className="flex flex-wrap gap-2">
                    {match.scorers.map((s, i) => (
                      <div key={i} className="bg-slate-100/60 border border-slate-200 rounded-full px-4 py-1.5 text-[11px] font-bold text-slate-600 flex items-center gap-2">
                        <span className="text-slate-800">{s.name}</span>
                        <span className="text-[9px] bg-white px-2 py-0.5 rounded-full text-green-700 border border-green-100 font-black">
                          {s.tries}T {s.conversions}C {s.penalties || 0}P
                        </span>
                      </div>
                    ))}
                  </div>
                </div>
              )}

              <div className="bg-slate-50/50 px-6 py-3 flex justify-between items-center text-[10px] text-slate-400 border-t border-slate-100">
                <div className="flex items-center gap-4">
                  <button onClick={() => startEdit(match)} className="text-blue-500 hover:text-blue-700 font-black uppercase tracking-widest flex items-center gap-1.5 transition-colors">
                    <Edit3 className="h-3 w-3" /> Edit
                  </button>
                  <button onClick={() => deleteMatch(match.id)} className="text-slate-400 hover:text-red-500 font-black uppercase tracking-widest flex items-center gap-1.5 transition-colors">
                    <Trash2 className="h-3 w-3" /> Delete
                  </button>
                </div>
                <span className="font-mono opacity-50 flex items-center gap-1"><Target className="h-3 w-3" /> {match.id.slice(0, 4)}</span>
              </div>
            </div>
          ))}
        </section>
      </main>

      <footer className="fixed bottom-0 left-0 right-0 bg-white/95 backdrop-blur-sm border-t border-slate-200 p-4 text-center text-[10px] text-slate-400 z-30 flex justify-center gap-6">
        <span className="font-bold text-green-700 uppercase">Magalieskruin 2026</span>
        <span className="flex items-center gap-1 uppercase">T=5 C=2 P=3</span>
        <span className="font-mono">ID: {user?.uid.slice(0, 8)}...</span>
      </footer>
    </div>
  );
}galieskruin-2026
