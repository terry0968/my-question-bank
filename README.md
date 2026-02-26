import React, { useState, useEffect, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInWithCustomToken, signInAnonymously, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, doc, addDoc, updateDoc, deleteDoc, onSnapshot, query, writeBatch } from 'firebase/firestore';
import { 
  Plus, 
  Search, 
  BookOpen, 
  Trash2, 
  Edit3, 
  CheckCircle, 
  ChevronRight, 
  Layout, 
  X,
  RefreshCw,
  Eye,
  EyeOff,
  UploadCloud,
  FileText,
  AlertCircle
} from 'lucide-react';

/**
 * --- 部署教學提示 ---
 * 1. 如果你在本地環境 (VS Code) 執行，請手動替換下方的 YOUR_... 資訊。
 * 2. 這些資訊可以在 Firebase Console -> 專案設定 中找到。
 */
const firebaseConfig = typeof __firebase_config !== 'undefined' 
  ? JSON.parse(__firebase_config) 
  : {
    apiKey: "YOUR_API_KEY",
    authDomain: "YOUR_PROJECT.firebaseapp.com",
    projectId: "YOUR_PROJECT_ID",
    storageBucket: "YOUR_PROJECT.appspot.com",
    messagingSenderId: "YOUR_SENDER_ID",
    appId: "YOUR_APP_ID"
  };

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'question-bank-pro';

export default function App() {
  const [user, setUser] = useState(null);
  const [questions, setQuestions] = useState([]);
  const [loading, setLoading] = useState(true);
  const [view, setView] = useState('list'); 
  const [searchTerm, setSearchTerm] = useState('');
  const [editingQuestion, setEditingQuestion] = useState(null);
  const [isModalOpen, setIsModalOpen] = useState(false);
  const [isImportOpen, setIsImportOpen] = useState(false);
  const [showAnswerMap, setShowAnswerMap] = useState({});

  // 表單與匯入狀態
  const [formData, setFormData] = useState({
    question: '',
    answer: '',
    explanation: '',
    category: '一般',
    difficulty: '中'
  });
  const [importText, setImportText] = useState('');
  const [importCategory, setImportCategory] = useState('未分類');
  const [importStatus, setImportStatus] = useState('');

  // 身份驗證初始化
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
    const unsubscribe = onAuthStateChanged(auth, setUser);
    return () => unsubscribe();
  }, []);

  // 同步 Firestore 數據
  useEffect(() => {
    if (!user) return;
    const q = query(collection(db, 'artifacts', appId, 'public', 'data', 'questions'));
    const unsubscribe = onSnapshot(q, (snapshot) => {
      const docs = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setQuestions(docs);
      setLoading(false);
    }, (error) => {
      console.error("Firestore sync error:", error);
      setLoading(false);
    });
    return () => unsubscribe();
  }, [user]);

  // 處理手動儲存
  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!user) return;
    const data = { ...formData, updatedAt: Date.now() };
    try {
      if (editingQuestion) {
        await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'questions', editingQuestion.id), data);
      } else {
        await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'questions'), { ...data, createdAt: Date.now() });
      }
      closeModal();
    } catch (err) {
      console.error("Save error:", err);
    }
  };

  // AI 批量解析匯入邏輯
  const handleBatchImport = async () => {
    if (!importText.trim() || !user) return;
    setImportStatus('正在智能解析並上傳中...');
    
    const lines = importText.split('\n').filter(l => l.trim() !== '');
    const batchQuestions = [];
    let currentQ = null;

    lines.forEach(line => {
      // 匹配題號開頭 (例如 1. 或 (1))
      if (line.match(/^(\d+[\.\、\s]|\(\d+\))/)) {
        if (currentQ) batchQuestions.push(currentQ);
        currentQ = { 
          question: line.replace(/^(\d+[\.\、\s]|\(\d+\))\s*/, '').trim(), 
          answer: '未設定', 
          explanation: '', 
          category: importCategory, 
          difficulty: '中' 
        };
      } else if (line.match(/(答|Ans|答案|解)[：:]/i)) {
        if (currentQ) currentQ.answer = line.replace(/.*(答|Ans|答案|解)[：:]/i, '').trim();
      } else if (line.match(/(解析|說明|Exp)[：:]/i)) {
        if (currentQ) currentQ.explanation = line.replace(/.*(解析|說明|Exp)[：:]/i, '').trim();
      } else if (currentQ && !currentQ.answer.includes(line)) {
        // 如果不是題號也不是答案，可能是題目換行內容
        currentQ.question += '\n' + line;
      }
    });
    if (currentQ) batchQuestions.push(currentQ);

    if (batchQuestions.length === 0) {
      setImportStatus('無法辨識題目，請檢查格式');
      return;
    }

    try {
      const batch = writeBatch(db);
      batchQuestions.forEach(q => {
        const newDocRef = doc(collection(db, 'artifacts', appId, 'public', 'data', 'questions'));
        batch.set(newDocRef, { ...q, createdAt: Date.now(), updatedAt: Date.now() });
      });
      await batch.commit();
      setImportStatus(`成功匯入 ${batchQuestions.length} 題！`);
      setTimeout(() => {
        setIsImportOpen(false);
        setImportText('');
        setImportStatus('');
      }, 2000);
    } catch (error) {
      setImportStatus('匯入失敗，請確認 Firebase 權限設定');
    }
  };

  const closeModal = () => { setIsModalOpen(false); setEditingQuestion(null); };

  const filteredQuestions = useMemo(() => {
    return questions.filter(q => q.question.toLowerCase().includes(searchTerm.toLowerCase()) || q.category.toLowerCase().includes(searchTerm.toLowerCase()))
      .sort((a, b) => (b.updatedAt || 0) - (a.updatedAt || 0));
  }, [questions, searchTerm]);

  // 測驗邏輯
  const [quizQuestions, setQuizQuestions] = useState([]);
  const [currentQuizIndex, setCurrentQuizIndex] = useState(0);
  const [showQuizAnswer, setShowQuizAnswer] = useState(false);

  const startQuiz = () => {
    if (questions.length === 0) return;
    const shuffled = [...questions].sort(() => 0.5 - Math.random()).slice(0, 10);
    setQuizQuestions(shuffled);
    setCurrentQuizIndex(0);
    setShowQuizAnswer(false);
    setView('quiz');
  };

  return (
    <div className="min-h-screen bg-slate-50 flex flex-col md:flex-row font-sans text-slate-900">
      
      {/* 側邊導航 */}
      <aside className="w-full md:w-64 bg-slate-900 text-white flex-shrink-0 flex flex-col md:h-screen sticky top-0 z-20">
        <div className="p-6 flex items-center gap-3">
          <div className="bg-blue-600 p-2 rounded-lg shadow-lg shadow-blue-900/20"><BookOpen size={24} /></div>
          <h1 className="text-xl font-black tracking-tighter">雲端題庫 Pro</h1>
        </div>
        <nav className="flex-1 px-4 py-2 space-y-2 flex flex-col">
          <button onClick={() => setView('list')} className={`flex items-center gap-3 px-4 py-3 rounded-xl transition-all ${view === 'list' ? 'bg-blue-600 shadow-lg' : 'hover:bg-slate-800 text-slate-400'}`}>
            <Layout size={20} /> <span className="font-bold">題庫總覽</span>
          </button>
          <button onClick={startQuiz} className={`flex items-center gap-3 px-4 py-3 rounded-xl transition-all ${view === 'quiz' ? 'bg-blue-600 shadow-lg' : 'hover:bg-slate-800 text-slate-400'}`}>
            <RefreshCw size={20} /> <span className="font-bold">隨機測驗</span>
          </button>
          
          <div className="pt-6 mt-4 border-t border-slate-800">
            <button onClick={() => setIsImportOpen(true)} className="w-full flex items-center gap-3 px-4 py-3 rounded-xl text-emerald-400 hover:bg-emerald-500/10 transition-colors border border-emerald-500/20">
              <UploadCloud size={20} /> <span className="font-bold">AI 批量匯入</span>
            </button>
          </div>
        </nav>
        
        <div className="p-4 m-4 bg-slate-800/50 rounded-xl">
          <p className="text-[10px] text-slate-500 font-mono break-all leading-tight">
            User: {user?.uid || '正在連接...'}
          </p>
        </div>
      </aside>

      <main className="flex-1 flex flex-col h-screen overflow-hidden">
        {/* 頂部搜尋欄 */}
        <header className="bg-white border-b px-6 py-4 flex items-center justify-between shadow-sm z-10">
          <div className="relative max-w-md w-full">
            <Search className="absolute left-3 top-1/2 -translate-y-1/2 text-slate-400" size={18} />
            <input 
              type="text" 
              placeholder="搜尋關鍵字、解析或分類..." 
              className="w-full pl-10 pr-4 py-2.5 bg-slate-100 rounded-full text-sm outline-none focus:ring-2 focus:ring-blue-500 transition-all border-none" 
              value={searchTerm} 
              onChange={e => setSearchTerm(e.target.value)} 
            />
          </div>
          <button 
            onClick={() => { setEditingQuestion(null); setFormData({question: '', answer: '', explanation: '', category: '一般', difficulty: '中'}); setIsModalOpen(true); }} 
            className="ml-4 bg-blue-600 hover:bg-blue-700 text-white px-6 py-2.5 rounded-full flex items-center gap-2 text-sm font-black shadow-lg shadow-blue-200 active:scale-95 transition-all"
          >
            <Plus size={18} /> <span>手動新增</span>
          </button>
        </header>

        {/* 內容顯示區 */}
        <div className="flex-1 overflow-y-auto p-4 md:p-8 bg-slate-50">
          {loading ? (
            <div className="flex flex-col items-center justify-center h-full text-slate-400">
              <RefreshCw className="animate-spin mb-4" size={32} />
              <p className="font-bold">雲端資料同步中...</p>
            </div>
          ) : view === 'list' ? (
            <div className="max-w-4xl mx-auto space-y-6">
              <div className="flex items-center justify-between mb-4">
                <h2 className="text-2xl font-black text-slate-800 tracking-tight">題庫清單 ({filteredQuestions.length})</h2>
              </div>

              {filteredQuestions.length === 0 ? (
                <div className="text-center py-24 bg-white rounded-[2rem] border-2 border-dashed border-slate-200">
                  <FileText className="mx-auto text-slate-200 mb-6" size={64} />
                  <p className="text-slate-400 font-bold">目前沒有符合條件的題目，嘗試新增或匯入吧！</p>
                </div>
              ) : (
                <div className="grid gap-4">
                  {filteredQuestions.map(q => (
                    <div key={q.id} className="bg-white p-6 rounded-3xl shadow-sm border border-slate-100 hover:shadow-xl hover:shadow-blue-900/5 transition-all group relative overflow-hidden">
                      <div className="flex justify-between items-start mb-4">
                        <div className="flex flex-wrap gap-2">
                          <span className="px-3 py-1 bg-blue-50 text-blue-600 text-[10px] font-black rounded-lg uppercase tracking-wider border border-blue-100">{q.category}</span>
                          <span className={`px-3 py-1 text-[10px] font-black rounded-lg border ${
                            q.difficulty === '高' ? 'bg-red-50 text-red-600 border-red-100' : 
                            q.difficulty === '低' ? 'bg-emerald-50 text-emerald-600 border-emerald-100' : 'bg-orange-50 text-orange-600 border-orange-100'
                          }`}>
                            難度：{q.difficulty}
                          </span>
                        </div>
                        <div className="flex gap-2 opacity-0 group-hover:opacity-100 transition-all">
                          <button onClick={() => { setEditingQuestion(q); setFormData(q); setIsModalOpen(true); }} className="p-2 bg-slate-50 hover:bg-blue-50 text-slate-400 hover:text-blue-600 rounded-xl transition-colors"><Edit3 size={16} /></button>
                          <button onClick={async () => { if(window.confirm('確定要永久刪除此題目嗎？')) await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'questions', q.id)) }} className="p-2 bg-slate-50 hover:bg-red-50 text-slate-400 hover:text-red-600 rounded-xl transition-colors"><Trash2 size={16} /></button>
                        </div>
                      </div>
                      <p className="text-lg font-bold text-slate-800 mb-6 leading-relaxed whitespace-pre-wrap">{q.question}</p>
                      
                      <div className="pt-4 border-t border-slate-50 flex items-center justify-between">
                        <button onClick={() => setShowAnswerMap(p => ({ ...p, [q.id]: !p[q.id] }))} className="flex items-center gap-2 text-sm font-black text-blue-600 hover:text-blue-800 transition-colors">
                          {showAnswerMap[q.id] ? <EyeOff size={18} /> : <Eye size={18} />} 
                          {showAnswerMap[q.id] ? '隱藏解答' : '顯示解答'}
                        </button>
                      </div>

                      {showAnswerMap[q.id] && (
                        <div className="mt-4 p-5 bg-slate-50 rounded-2xl border border-slate-100 animate-in fade-in slide-in-from-top-2">
                          <div className="flex items-start gap-2 text-emerald-600 font-black mb-2">
                            <CheckCircle size={20} className="shrink-0 mt-0.5" />
                            <span className="text-lg">答案：{q.answer}</span>
                          </div>
                          {q.explanation && (
                            <div className="text-sm text-slate-500 leading-relaxed border-l-4 border-slate-200 pl-4 ml-2 mt-3">
                              <span className="font-black text-slate-400 block mb-1 uppercase text-[10px]">Explanation</span>
                              {q.explanation}
                            </div>
                          )}
                        </div>
                      )}
                    </div>
                  ))}
                </div>
              )}
            </div>
          ) : view === 'quiz' ? (
             <div className="max-w-2xl mx-auto py-10">
                <div className="bg-white rounded-[2.5rem] shadow-2xl overflow-hidden border border-slate-100">
                  <div className="bg-slate-900 p-8 text-white flex justify-between items-center">
                    <div>
                      <h3 className="text-xs font-black uppercase tracking-[0.2em] text-blue-400 mb-1">Testing Mode</h3>
                      <p className="font-bold">隨機實戰測驗中</p>
                    </div>
                    <span className="bg-blue-600 px-5 py-2 rounded-2xl text-xs font-black shadow-lg">第 {currentQuizIndex + 1} / {quizQuestions.length} 題</span>
                  </div>
                  
                  <div className="p-12 text-center min-h-[350px] flex flex-col justify-center items-center">
                    <h2 className="text-2xl font-black mb-12 text-slate-800 leading-tight">{quizQuestions[currentQuizIndex]?.question}</h2>
                    
                    {showQuizAnswer ? (
                      <div className="w-full bg-blue-50/50 border border-blue-100 p-8 rounded-[2rem] animate-in zoom-in duration-300">
                        <p className="text-blue-600 font-black text-xs uppercase tracking-widest mb-3">Correct Answer</p>
                        <p className="text-4xl font-black text-slate-900 mb-6">{quizQuestions[currentQuizIndex]?.answer}</p>
                        {quizQuestions[currentQuizIndex]?.explanation && (
                          <div className="text-slate-500 text-sm italic border-t border-blue-100 pt-6 leading-relaxed">
                            {quizQuestions[currentQuizIndex]?.explanation}
                          </div>
                        )}
                      </div>
                    ) : (
                      <button 
                        onClick={() => setShowQuizAnswer(true)} 
                        className="group relative bg-slate-900 text-white px-12 py-5 rounded-full font-black text-lg shadow-2xl hover:bg-blue-600 transition-all active:scale-95 overflow-hidden"
                      >
                        <span className="relative z-10">揭曉標準答案</span>
                        <div className="absolute inset-0 bg-gradient-to-r from-blue-600 to-indigo-600 opacity-0 group-hover:opacity-100 transition-opacity"></div>
                      </button>
                    )}
                  </div>
                  
                  <div className="p-8 bg-slate-50 flex gap-4">
                    <button onClick={() => setView('list')} className="flex-1 font-black text-slate-400 hover:text-slate-600 transition-colors uppercase text-xs tracking-widest">Quit Quiz</button>
                    <button 
                      disabled={currentQuizIndex === quizQuestions.length - 1} 
                      onClick={() => { setCurrentQuizIndex(i => i + 1); setShowQuizAnswer(false); }} 
                      className={`flex-[2] py-4 rounded-2xl font-black shadow-lg transition-all flex items-center justify-center gap-2 ${
                        currentQuizIndex === quizQuestions.length - 1 
                        ? 'bg-slate-200 text-slate-400 cursor-not-allowed' 
                        : 'bg-blue-600 text-white hover:bg-blue-700 active:translate-x-1'
                      }`}
                    >
                      下一題 <ChevronRight size={20} />
                    </button>
                  </div>
                </div>
             </div>
          ) : null}
        </div>
      </main>

      {/* 批量匯入視窗 */}
      {isImportOpen && (
        <div className="fixed inset-0 z-50 bg-slate-900/90 backdrop-blur-md flex items-center justify-center p-4 animate-in fade-in duration-300">
          <div className="bg-white w-full max-w-2xl rounded-[3rem] shadow-2xl flex flex-col max-h-[90vh] overflow-hidden animate-in zoom-in-95 duration-300">
            <div className="p-10 border-b flex justify-between items-center bg-slate-50/50">
              <div>
                <h3 className="text-3xl font-black text-slate-800 tracking-tighter">AI 批量匯入</h3>
                <p className="text-slate-400 text-sm font-medium mt-1">智能解析 PDF 或考卷內容並同步雲端</p>
              </div>
              <button onClick={() => setIsImportOpen(false)} className="p-4 hover:bg-slate-200 text-slate-400 hover:text-slate-800 rounded-full transition-all"><X size={28} /></button>
            </div>
            
            <div className="p-10 space-y-6 overflow-y-auto">
              <div className="bg-blue-50 border border-blue-100 p-6 rounded-[1.5rem] flex gap-4 text-blue-700 text-xs leading-relaxed">
                <AlertCircle size={24} className="shrink-0 text-blue-500" />
                <div>
                  <p className="font-black mb-1 uppercase tracking-wider">Format Helper 格式提示</p>
                  <p>請確保每題開頭有數字（如 1.），答案請用「答：」或「Ans:」標註。系統會自動辨識並分割每一道考題。</p>
                </div>
              </div>
              
              <div>
                <label className="block text-[10px] font-black text-slate-400 uppercase tracking-[0.2em] mb-3 ml-2">Category 分類設定</label>
                <input 
                  type="text" 
                  placeholder="例如：生成式 AI 證照、期末考、112年考古題" 
                  className="w-full px-6 py-4 rounded-2xl border-2 border-slate-100 outline-none focus:border-emerald-500 font-bold transition-all" 
                  value={importCategory} 
                  onChange={e => setImportCategory(e.target.value)} 
                />
              </div>
              
              <div>
                <label className="block text-[10px] font-black text-slate-400 uppercase tracking-[0.2em] mb-3 ml-2">Content 貼上內容</label>
                <textarea 
                  rows="10" 
                  className="w-full px-6 py-5 rounded-2xl border-2 border-slate-100 outline-none focus:border-emerald-500 font-medium text-sm leading-relaxed" 
                  placeholder="範例內容：&#10;1. 請問 AI 的全名是什麼？&#10;答：Artificial Intelligence。&#10;解析：這是一門電腦科學的分支..." 
                  value={importText} 
                  onChange={e => setImportText(e.target.value)}
                ></textarea>
              </div>
              
              {importStatus && (
                <div className="text-center font-black text-emerald-600 animate-bounce py-2">
                  {importStatus}
                </div>
              )}
              
              <button 
                onClick={handleBatchImport} 
                disabled={!importText.trim()} 
                className="group w-full bg-emerald-600 text-white py-6 rounded-[1.5rem] font-black text-xl hover:bg-emerald-700 shadow-2xl shadow-emerald-200 disabled:opacity-30 disabled:shadow-none transition-all active:scale-95"
              >
                開始自動解析並同步雲端
              </button>
            </div>
          </div>
        </div>
      )}

      {/* 手動編輯/新增視窗 */}
      {isModalOpen && (
        <div className="fixed inset-0 z-50 bg-slate-900/80 backdrop-blur-md flex items-center justify-center p-4 animate-in fade-in duration-200">
          <form onSubmit={handleSubmit} className="bg-white w-full max-w-lg rounded-[2.5rem] shadow-2xl p-10 space-y-6 animate-in zoom-in-95 duration-300">
            <div className="flex justify-between items-center mb-6">
              <h3 className="text-2xl font-black text-slate-800">{editingQuestion ? '修改考題' : '建立新考題'}</h3>
              <button type="button" onClick={closeModal} className="p-2 hover:bg-slate-100 rounded-full transition-colors"><X size={24} /></button>
            </div>
            
            <div>
              <label className="block text-[10px] font-black text-slate-400 uppercase tracking-widest mb-2 ml-2">Question 題目內容</label>
              <textarea 
                required 
                rows="3" 
                className="w-full px-6 py-4 rounded-2xl border-2 border-slate-100 outline-none focus:border-blue-500 font-bold text-slate-700 transition-all" 
                placeholder="請輸入題目敘述..." 
                value={formData.question} 
                onChange={e => setFormData({...formData, question: e.target.value})}
              ></textarea>
            </div>

            <div className="grid grid-cols-2 gap-6">
              <div>
                <label className="block text-[10px] font-black text-slate-400 uppercase tracking-widest mb-2 ml-2">Category 分類</label>
                <input type="text" className="w-full px-6 py-3 rounded-2xl border-2 border-slate-100 outline-none focus:border-blue-500 font-bold" placeholder="如：數學" value={formData.category} onChange={e => setFormData({...formData, category: e.target.value})} />
              </div>
              <div>
                <label className="block text-[10px] font-black text-slate-400 uppercase tracking-widest mb-2 ml-2">Difficulty 難度</label>
                <select className="w-full px-6 py-3 rounded-2xl border-2 border-slate-100 outline-none font-bold bg-white" value={formData.difficulty} onChange={e => setFormData({...formData, difficulty: e.target.value})}>
                  <option value="低">簡單 (Easy)</option>
                  <option value="中">普通 (Medium)</option>
                  <option value="高">困難 (Hard)</option>
                </select>
              </div>
            </div>

            <div>
              <label className="block text-[10px] font-black text-slate-400 uppercase tracking-widest mb-2 ml-2">Answer 標準答案</label>
              <input required type="text" className="w-full px-6 py-4 rounded-2xl border-2 border-slate-100 outline-none font-black text-blue-600 text-lg focus:border-blue-500" placeholder="輸入正確答案" value={formData.answer} onChange={e => setFormData({...formData, answer: e.target.value})} />
            </div>

            <div>
              <label className="block text-[10px] font-black text-slate-400 uppercase tracking-widest mb-2 ml-2">Explanation 補充解析</label>
              <textarea rows="2" className="w-full px-6 py-3 rounded-2xl border-2 border-slate-100 outline-none text-sm text-slate-500" placeholder="填寫解題思路或重點..." value={formData.explanation} onChange={e => setFormData({...formData, explanation: e.target.value})}></textarea>
            </div>

            <button type="submit" className="w-full bg-slate-900 text-white py-5 rounded-2xl font-black text-xl shadow-2xl hover:bg-blue-600 transition-all active:scale-95 mt-4">
              確認儲存並發佈
            </button>
          </form>
        </div>
      )}
    </div>
  );
}
