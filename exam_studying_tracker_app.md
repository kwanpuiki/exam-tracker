<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>AceTracker - Exam Studying Suite</title>
  
  <!-- Tailwind CSS CDN -->
  <script src="https://cdn.tailwindcss.com"></script>
  
  <!-- React & ReactDOM CDNs -->
  <script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
  <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
  
  <!-- Babel CDN for inline JSX transformation -->
  <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
  
  <!-- Lucide Icons CDN -->
  <script src="https://unpkg.com/lucide@latest"></script>
</head>
<body class="bg-slate-950 text-slate-100 font-sans antialiased selection:bg-indigo-500 selection:text-white">
  <div id="root"></div>

  <script type="text/babel">
    const { useState, useEffect, useRef, useMemo } = React;

    // --- INITIAL MOCK DATA ---
    const INITIAL_EXAMS = [
      {
        id: 'exam-1',
        subject: 'Computer Science',
        title: 'Data Structures & Algorithms Final',
        date: new Date(Date.now() + 8 * 24 * 60 * 60 * 1000).toISOString().split('T')[0],
        time: '09:00',
        targetGrade: 'A (95%)',
        priority: 'High',
        color: '#6366f1',
        status: 'Upcoming'
      },
      {
        id: 'exam-2',
        subject: 'Organic Chemistry',
        title: 'Midterm Examination',
        date: new Date(Date.now() + 3 * 24 * 60 * 60 * 1000).toISOString().split('T')[0],
        time: '14:00',
        targetGrade: 'A-',
        priority: 'High',
        color: '#ec4899',
        status: 'Upcoming'
      },
      {
        id: 'exam-3',
        subject: 'Linear Algebra',
        title: 'Matrices & Vector Spaces Test',
        date: new Date(Date.now() + 14 * 24 * 60 * 60 * 1000).toISOString().split('T')[0],
        time: '11:30',
        targetGrade: 'B+',
        priority: 'Medium',
        color: '#3b82f6',
        status: 'Upcoming'
      }
    ];

    const INITIAL_TOPICS = [
      { id: 't-1', examId: 'exam-1', name: 'Binary Search Trees & AVL Trees', status: 'Mastered' },
      { id: 't-2', examId: 'exam-1', name: 'Graph Algorithms (Dijkstra, A*)', status: 'In Progress' },
      { id: 't-3', examId: 'exam-1', name: 'Dynamic Programming Patterns', status: 'Reviewed' },
      { id: 't-4', examId: 'exam-1', name: 'Sorting Algorithms Complexity', status: 'Mastered' },
      { id: 't-5', examId: 'exam-1', name: 'System Design Basics', status: 'Not Started' },
      { id: 't-6', examId: 'exam-2', name: 'Electrophilic Addition Reactions', status: 'Reviewed' },
      { id: 't-7', examId: 'exam-2', name: 'Stereochemistry & Chirality', status: 'Mastered' },
      { id: 't-8', examId: 'exam-2', name: 'NMR Spectroscopy Interpretation', status: 'In Progress' },
      { id: 't-9', examId: 'exam-2', name: 'Aromaticity & Hückel\'s Rule', status: 'Not Started' },
      { id: 't-10', examId: 'exam-3', name: 'Eigenvalues & Eigenvectors', status: 'In Progress' },
      { id: 't-11', examId: 'exam-3', name: 'Orthogonalization & Gram-Schmidt', status: 'Not Started' },
      { id: 't-12', examId: 'exam-3', name: 'Matrix Transformations', status: 'Reviewed' }
    ];

    const INITIAL_SESSIONS = [
      { id: 's-1', examId: 'exam-1', topicId: 't-1', durationMinutes: 50, timestamp: new Date(Date.now() - 2 * 24 * 60 * 60 * 1000).toISOString() },
      { id: 's-2', examId: 'exam-1', topicId: 't-2', durationMinutes: 25, timestamp: new Date(Date.now() - 1 * 24 * 60 * 60 * 1000).toISOString() },
      { id: 's-3', examId: 'exam-2', topicId: 't-6', durationMinutes: 50, timestamp: new Date(Date.now() - 1 * 24 * 60 * 60 * 1000).toISOString() },
      { id: 's-4', examId: 'exam-2', topicId: 't-7', durationMinutes: 75, timestamp: new Date().toISOString() },
      { id: 's-5', examId: 'exam-3', topicId: 't-10', durationMinutes: 25, timestamp: new Date().toISOString() }
    ];

    // Sound chime helper using Web Audio API
    const playChimeSound = () => {
      try {
        const AudioContext = window.AudioContext || window.webkitAudioContext;
        if (!AudioContext) return;
        const ctx = new AudioContext();
        const now = ctx.currentTime;
        
        [523.25, 659.25, 783.99].forEach((freq, index) => {
          const osc = ctx.createOscillator();
          const gain = ctx.createGain();
          osc.type = 'sine';
          osc.frequency.setValueAtTime(freq, now + index * 0.12);
          
          gain.gain.setValueAtTime(0, now + index * 0.12);
          gain.gain.linearRampToValueAtTime(0.2, now + index * 0.12 + 0.05);
          gain.gain.exponentialRampToValueAtTime(0.001, now + index * 0.12 + 1.2);
          
          osc.connect(gain);
          gain.connect(ctx.destination);
          
          osc.start(now + index * 0.12);
          osc.stop(now + index * 0.12 + 1.3);
        });
      } catch (e) {
        console.error("Audio error", e);
      }
    };

    function App() {
      const [activeTab, setActiveTab] = useState('dashboard');
      const [exams, setExams] = useState(() => {
        const saved = localStorage.getItem('est_exams');
        return saved ? JSON.parse(saved) : INITIAL_EXAMS;
      });
      const [topics, setTopics] = useState(() => {
        const saved = localStorage.getItem('est_topics');
        return saved ? JSON.parse(saved) : INITIAL_TOPICS;
      });
      const [studySessions, setStudySessions] = useState(() => {
        const saved = localStorage.getItem('est_sessions');
        return saved ? JSON.parse(saved) : INITIAL_SESSIONS;
      });

      const [dailyGoalMinutes] = useState(120);
      const [soundEnabled, setSoundEnabled] = useState(true);
      const [isExamModalOpen, setIsExamModalOpen] = useState(false);
      const [editingExam, setEditingExam] = useState(null);

      useEffect(() => {
        localStorage.setItem('est_exams', JSON.stringify(exams));
      }, [exams]);

      useEffect(() => {
        localStorage.setItem('est_topics', JSON.stringify(topics));
      }, [topics]);

      useEffect(() => {
        localStorage.setItem('est_sessions', JSON.stringify(studySessions));
      }, [studySessions]);

      useEffect(() => {
        if (window.lucide) {
          window.lucide.createIcons();
        }
      });

      const todayMinutesStudied = useMemo(() => {
        const todayStr = new Date().toISOString().split('T')[0];
        return studySessions
          .filter(s => new Date(s.timestamp).toISOString().split('T')[0] === todayStr)
          .reduce((acc, curr) => acc + curr.durationMinutes, 0);
      }, [studySessions]);

      const streakDays = useMemo(() => {
        if (studySessions.length === 0) return 0;
        const dates = Array.from(
          new Set(studySessions.map(s => new Date(s.timestamp).toISOString().split('T')[0]))
        ).sort((a, b) => new Date(b) - new Date(a));

        const today = new Date().toISOString().split('T')[0];
        const yesterday = new Date(Date.now() - 86400000).toISOString().split('T')[0];

        let streak = 0;
        let checkDate = new Date();

        if (dates.includes(today)) {
          streak++;
          checkDate.setDate(checkDate.getDate() - 1);
        } else if (dates.includes(yesterday)) {
          checkDate.setDate(checkDate.getDate() - 1);
        } else {
          return 0;
        }

        while (true) {
          const dateStr = checkDate.toISOString().split('T')[0];
          if (dates.includes(dateStr)) {
            if (dateStr !== today) streak++;
            checkDate.setDate(checkDate.getDate() - 1);
          } else {
            break;
          }
        }
        return streak;
      }, [studySessions]);

      const handleSaveExam = (examData) => {
        if (examData.id) {
          setExams(exams.map(e => e.id === examData.id ? examData : e));
        } else {
          const newExam = {
            ...examData,
            id: 'exam-' + Date.now(),
            status: 'Upcoming'
          };
          setExams([...exams, newExam]);
        }
        setIsExamModalOpen(false);
        setEditingExam(null);
      };

      const handleDeleteExam = (examId) => {
        setExams(exams.filter(e => e.id !== examId));
        setTopics(topics.filter(t => t.examId !== examId));
      };

      const handleAddTopic = (topicName, examId) => {
        if (!topicName.trim() || !examId) return;
        const newTopic = {
          id: 'topic-' + Date.now(),
          examId,
          name: topicName.trim(),
          status: 'Not Started'
        };
        setTopics([...topics, newTopic]);
      };

      const handleUpdateTopicStatus = (topicId, newStatus) => {
        setTopics(topics.map(t => t.id === topicId ? { ...t, status: newStatus } : t));
      };

      const handleDeleteTopic = (topicId) => {
        setTopics(topics.filter(t => t.id !== topicId));
      };

      const handleLogSession = (sessionData) => {
        const newSession = {
          id: 'sess-' + Date.now(),
          ...sessionData,
          timestamp: new Date().toISOString()
        };
        setStudySessions([newSession, ...studySessions]);
      };

      return (
        <div className="flex h-screen bg-slate-950 text-slate-100 font-sans overflow-hidden">
          {/* SIDEBAR NAVIGATION */}
          <aside className="w-64 bg-slate-900 border-r border-slate-800 flex flex-col justify-between hidden md:flex shrink-0">
            <div>
              <div className="p-6 flex items-center space-x-3 border-b border-slate-800/80">
                <div className="p-2.5 bg-gradient-to-tr from-indigo-600 to-violet-500 rounded-xl shadow-lg shadow-indigo-500/20 text-white">
                  <i data-lucide="sparkles" className="w-6 h-6"></i>
                </div>
                <div>
                  <h1 className="font-bold text-lg text-white tracking-wide">AceTracker</h1>
                  <p className="text-xs text-slate-400">Exam Study Suite</p>
                </div>
              </div>

              <nav className="p-4 space-y-1.5">
                <NavItem icon="layout-dashboard" label="Dashboard" active={activeTab === 'dashboard'} onClick={() => setActiveTab('dashboard')} />
                <NavItem icon="calendar" label="Exams & Schedule" active={activeTab === 'exams'} onClick={() => setActiveTab('exams')} badge={exams.length} />
                <NavItem icon="layers" label="Syllabus & Topics" active={activeTab === 'syllabus'} onClick={() => setActiveTab('syllabus')} />
                <NavItem icon="clock" label="Pomodoro Timer" active={activeTab === 'timer'} onClick={() => setActiveTab('timer')} />
                <NavItem icon="bar-chart-2" label="Study Analytics" active={activeTab === 'analytics'} onClick={() => setActiveTab('analytics')} />
              </nav>
            </div>

            <div className="p-4 border-t border-slate-800/80 m-3 rounded-2xl bg-slate-800/40 border">
              <div className="flex items-center justify-between mb-2">
                <span className="text-xs font-semibold text-slate-400">DAILY STREAK</span>
                <div className="flex items-center space-x-1 text-amber-400 font-bold text-sm">
                  <i data-lucide="flame" className="w-4 h-4 text-amber-400"></i>
                  <span>{streakDays} Days</span>
                </div>
              </div>
              <div className="w-full bg-slate-700/50 rounded-full h-1.5 overflow-hidden">
                <div 
                  className="bg-amber-400 h-1.5 rounded-full transition-all duration-500" 
                  style={{ width: `${Math.min(100, (todayMinutesStudied / dailyGoalMinutes) * 100)}%` }}
                />
              </div>
              <p className="text-[11px] text-slate-400 mt-2 text-right">
                {(todayMinutesStudied / 60).toFixed(1)} / {(dailyGoalMinutes / 60).toFixed(1)} hrs today
              </p>
            </div>
          </aside>

          {/* MAIN CONTENT AREA */}
          <div className="flex-1 flex flex-col min-w-0 overflow-hidden">
            <header className="h-16 border-b border-slate-800 bg-slate-900/50 backdrop-blur-md px-6 flex items-center justify-between shrink-0">
              <div className="flex items-center space-x-3 md:hidden">
                <div className="p-2 bg-indigo-600 rounded-lg text-white">
                  <i data-lucide="sparkles" className="w-5 h-5"></i>
                </div>
                <span className="font-bold text-lg text-white">AceTracker</span>
              </div>

              <div className="hidden md:flex items-center space-x-2 text-sm text-slate-400">
                <span className="capitalize font-medium text-slate-200">{activeTab}</span>
                <span>/</span>
                <span>Overview</span>
              </div>

              <div className="flex items-center space-x-3">
                <button 
                  onClick={() => setSoundEnabled(!soundEnabled)} 
                  className={`p-2 rounded-lg border border-slate-700 transition-colors ${soundEnabled ? 'bg-slate-800 text-slate-200' : 'bg-slate-900 text-slate-500'}`}
                  title={soundEnabled ? "Sound Effects On" : "Sound Effects Off"}
                >
                  <i data-lucide={soundEnabled ? "volume-2" : "volume-x"} className="w-4 h-4"></i>
                </button>

                <button 
                  onClick={() => {
                    setEditingExam(null);
                    setIsExamModalOpen(true);
                  }}
                  className="flex items-center space-x-2 bg-indigo-600 hover:bg-indigo-500 text-white px-3.5 py-1.5 rounded-lg text-sm font-medium transition-all shadow-md shadow-indigo-600/20"
                >
                  <i data-lucide="plus" className="w-4 h-4"></i>
                  <span>Add Exam</span>
                </button>
              </div>
            </header>

            {/* Mobile Tab Header */}
            <div className="md:hidden flex border-b border-slate-800 bg-slate-900 overflow-x-auto text-xs py-2 px-3 space-x-2">
              {['dashboard', 'exams', 'syllabus', 'timer', 'analytics'].map((tab) => (
                <button
                  key={tab}
                  onClick={() => setActiveTab(tab)}
                  className={`px-3 py-1.5 rounded-md whitespace-nowrap capitalize font-medium transition-all ${
                    activeTab === tab ? 'bg-indigo-600 text-white' : 'text-slate-400 hover:text-slate-200'
                  }`}
                >
                  {tab}
                </button>
              ))}
            </div>

            <main className="flex-1 overflow-y-auto p-4 sm:p-6 lg:p-8">
              {activeTab === 'dashboard' && (
                <DashboardView 
                  exams={exams} 
                  topics={topics} 
                  studySessions={studySessions} 
                  setActiveTab={setActiveTab}
                  todayMinutesStudied={todayMinutesStudied}
                  dailyGoalMinutes={dailyGoalMinutes}
                  streakDays={streakDays}
                />
              )}

              {activeTab === 'exams' && (
                <ExamsView 
                  exams={exams} 
                  topics={topics}
                  onAddExam={() => { setEditingExam(null); setIsExamModalOpen(true); }}
                  onEditExam={(exam) => { setEditingExam(exam); setIsExamModalOpen(true); }}
                  onDeleteExam={handleDeleteExam}
                />
              )}

              {activeTab === 'syllabus' && (
                <SyllabusView 
                  exams={exams} 
                  topics={topics}
                  onAddTopic={handleAddTopic}
                  onUpdateStatus={handleUpdateTopicStatus}
                  onDeleteTopic={handleDeleteTopic}
                />
              )}

              {activeTab === 'timer' && (
                <PomodoroTimerView 
                  exams={exams}
                  topics={topics}
                  onLogSession={handleLogSession}
                  soundEnabled={soundEnabled}
                />
              )}

              {activeTab === 'analytics' && (
                <AnalyticsView 
                  exams={exams}
                  topics={topics}
                  studySessions={studySessions}
                />
              )}
            </main>
          </div>

          {isExamModalOpen && (
            <ExamModal 
              exam={editingExam}
              onClose={() => setIsExamModalOpen(false)}
              onSave={handleSaveExam}
            />
          )}
        </div>
      );
    }

    function NavItem({ icon, label, active, onClick, badge }) {
      return (
        <button
          onClick={onClick}
          className={`w-full flex items-center justify-between px-3.5 py-2.5 rounded-xl font-medium text-sm transition-all duration-150 ${
            active 
              ? 'bg-indigo-600/10 text-indigo-400 border border-indigo-500/20' 
              : 'text-slate-400 hover:text-slate-200 hover:bg-slate-800/50'
          }`}
        >
          <div className="flex items-center space-x-3">
            <i data-lucide={icon} className="w-5 h-5"></i>
            <span>{label}</span>
          </div>
          {badge !== undefined && (
            <span className="text-xs bg-slate-800 text-slate-300 border border-slate-700 font-semibold px-2 py-0.5 rounded-full">
              {badge}
            </span>
          )}
        </button>
      );
    }

    // --- DASHBOARD VIEW ---
    function DashboardView({ exams, topics, studySessions, setActiveTab, todayMinutesStudied, dailyGoalMinutes, streakDays }) {
      const upcomingExams = useMemo(() => {
        return [...exams]
          .filter(e => e.status === 'Upcoming')
          .sort((a, b) => new Date(`${a.date}T${a.time || '00:00'}`) - new Date(`${b.date}T${b.time || '00:00'}`))
          .slice(0, 3);
      }, [exams]);

      const pendingRevisions = useMemo(() => {
        return topics.filter(t => t.status === 'In Progress' || t.status === 'Reviewed');
      }, [topics]);

      return (
        <div className="max-w-7xl mx-auto space-y-8">
          <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4">
            <StatCard icon="flame" label="Current Streak" value={`${streakDays} Days`} subtitle="Keep the momentum going!" bg="from-amber-500/10 to-orange-500/5" borderColor="border-amber-500/20" />
            <StatCard icon="clock" label="Studied Today" value={`${Math.floor(todayMinutesStudied / 60)}h ${todayMinutesStudied % 60}m`} subtitle={`Goal: ${(dailyGoalMinutes / 60).toFixed(1)} hours`} bg="from-indigo-500/10 to-blue-500/5" borderColor="border-indigo-500/20" />
            <StatCard icon="target" label="Upcoming Exams" value={upcomingExams.length} subtitle={upcomingExams[0] ? `Next in ${getDaysUntil(upcomingExams[0].date)} days` : 'No upcoming exams'} bg="from-emerald-500/10 to-teal-500/5" borderColor="border-emerald-500/20" />
            <StatCard icon="check-square" label="Topics Mastered" value={`${topics.filter(t => t.status === 'Mastered').length} / ${topics.length}`} subtitle={`${topics.length > 0 ? Math.round((topics.filter(t => t.status === 'Mastered').length / topics.length) * 100) : 0}% Mastered`} bg="from-purple-500/10 to-pink-500/5" borderColor="border-purple-500/20" />
          </div>

          <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
            <div className="lg:col-span-2 space-y-8">
              {upcomingExams.length > 0 ? (
                <div className="bg-gradient-to-r from-slate-900 via-slate-900/90 to-indigo-950/40 border border-slate-800 rounded-2xl p-6 shadow-xl relative overflow-hidden">
                  <div className="flex flex-wrap items-center justify-between gap-2 mb-4">
                    <span className="px-3 py-1 bg-indigo-500/20 border border-indigo-500/30 text-indigo-300 text-xs font-semibold rounded-full uppercase tracking-wider">
                      Highest Priority Exam
                    </span>
                    <span className="text-xs text-slate-400">
                      Target Grade: <strong className="text-slate-200">{upcomingExams[0].targetGrade}</strong>
                    </span>
                  </div>

                  <div className="mb-6">
                    <h2 className="text-2xl font-bold text-white">{upcomingExams[0].title}</h2>
                    <p className="text-slate-400 text-sm">{upcomingExams[0].subject}</p>
                  </div>

                  <CountdownTimer targetDate={`${upcomingExams[0].date}T${upcomingExams[0].time || '09:00'}`} />

                  <div className="mt-6 flex flex-wrap items-center justify-between gap-4 pt-4 border-t border-slate-800/80">
                    <div className="flex items-center space-x-2 text-slate-400 text-xs">
                      <i data-lucide="calendar" className="w-4 h-4 text-slate-500"></i>
                      <span>{new Date(upcomingExams[0].date).toLocaleDateString(undefined, { weekday: 'short', month: 'short', day: 'numeric' })}</span>
                      <span>•</span>
                      <span>{upcomingExams[0].time || '09:00'}</span>
                    </div>

                    <button 
                      onClick={() => setActiveTab('timer')}
                      className="flex items-center space-x-2 bg-indigo-600 hover:bg-indigo-500 text-white px-4 py-2 rounded-xl text-sm font-medium transition-all shadow-lg shadow-indigo-600/30"
                    >
                      <i data-lucide="play" className="w-4 h-4"></i>
                      <span>Start Studying Now</span>
                    </button>
                  </div>
                </div>
              ) : (
                <div className="bg-slate-900 border border-slate-800 rounded-2xl p-8 text-center">
                  <p className="text-slate-400">No upcoming exams scheduled!</p>
                  <button onClick={() => setActiveTab('exams')} className="mt-4 text-indigo-400 hover:underline text-sm font-medium">+ Add your first exam</button>
                </div>
              )}

              <div className="bg-slate-900 border border-slate-800 rounded-2xl p-6">
                <div className="flex items-center justify-between mb-6">
                  <h3 className="font-bold text-lg text-white">Subject Mastery Progress</h3>
                  <button onClick={() => setActiveTab('syllabus')} className="text-xs text-indigo-400 hover:text-indigo-300 font-medium flex items-center space-x-1">
                    <span>View Full Syllabus</span>
                    <i data-lucide="chevron-right" className="w-3.5 h-3.5"></i>
                  </button>
                </div>

                <div className="space-y-4">
                  {exams.map(exam => {
                    const examTopics = topics.filter(t => t.examId === exam.id);
                    const masteredCount = examTopics.filter(t => t.status === 'Mastered').length;
                    const pct = examTopics.length > 0 ? Math.round((masteredCount / examTopics.length) * 100) : 0;

                    return (
                      <div key={exam.id} className="p-4 bg-slate-950/60 border border-slate-800/80 rounded-xl space-y-2">
                        <div className="flex items-center justify-between">
                          <div className="flex items-center space-x-2">
                            <div className="w-3 h-3 rounded-full" style={{ backgroundColor: exam.color || '#6366f1' }} />
                            <span className="font-semibold text-sm text-slate-200">{exam.subject}</span>
                          </div>
                          <span className="text-xs font-mono text-slate-400">{pct}% Complete</span>
                        </div>
                        <div className="w-full bg-slate-800 rounded-full h-2 overflow-hidden">
                          <div className="h-2 rounded-full transition-all duration-500" style={{ width: `${pct}%`, backgroundColor: exam.color || '#6366f1' }} />
                        </div>
                      </div>
                    );
                  })}
                </div>
              </div>
            </div>

            <div className="space-y-8">
              <div className="bg-slate-900 border border-slate-800 rounded-2xl p-6">
                <div className="flex items-center space-x-2 mb-4">
                  <i data-lucide="rotate-ccw" className="w-5 h-5 text-indigo-400"></i>
                  <h3 className="font-bold text-lg text-white">Revision Reminders</h3>
                </div>
                <div className="space-y-3">
                  {pendingRevisions.length > 0 ? (
                    pendingRevisions.slice(0, 4).map(topic => {
                      const exam = exams.find(e => e.id === topic.examId);
                      return (
                        <div key={topic.id} className="p-3 bg-slate-950/80 border border-slate-800 rounded-xl flex items-center justify-between">
                          <div>
                            <p className="text-xs font-semibold text-slate-200">{topic.name}</p>
                            <p className="text-[10px] text-slate-400">{exam?.subject || 'General'}</p>
                          </div>
                          <span className="text-[10px] px-2 py-0.5 rounded font-medium bg-amber-500/10 text-amber-400 border border-amber-500/20">{topic.status}</span>
                        </div>
                      );
                    })
                  ) : (
                    <p className="text-xs text-slate-500 py-4 text-center">No pending revisions!</p>
                  )}
                </div>
              </div>
            </div>
          </div>
        </div>
      );
    }

    function StatCard({ icon, label, value, subtitle, bg, borderColor }) {
      return (
        <div className={`p-5 rounded-2xl bg-gradient-to-br ${bg} border ${borderColor} bg-slate-900/90 shadow-md`}>
          <div className="flex items-center justify-between mb-3">
            <span className="text-xs font-medium text-slate-400">{label}</span>
            <div className="p-2 rounded-xl bg-slate-900/60 border border-slate-800">
              <i data-lucide={icon} className="w-6 h-6"></i>
            </div>
          </div>
          <div className="text-2xl font-bold text-white mb-1">{value}</div>
          <p className="text-[11px] text-slate-400">{subtitle}</p>
        </div>
      );
    }

    function CountdownTimer({ targetDate }) {
      const [timeLeft, setTimeLeft] = useState({ days: 0, hours: 0, minutes: 0, seconds: 0 });

      useEffect(() => {
        const calculateTime = () => {
          const difference = +new Date(targetDate) - +new Date();
          if (difference > 0) {
            setTimeLeft({
              days: Math.floor(difference / (1000 * 60 * 60 * 24)),
              hours: Math.floor((difference / (1000 * 60 * 60)) % 24),
              minutes: Math.floor((difference / 1000 / 60) % 60),
              seconds: Math.floor((difference / 1000) % 60),
            });
          }
        };

        calculateTime();
        const timer = setInterval(calculateTime, 1000);
        return () => clearInterval(timer);
      }, [targetDate]);

      return (
        <div className="grid grid-cols-4 gap-2 sm:gap-4 my-4">
          {[
            { label: 'DAYS', val: timeLeft.days },
            { label: 'HOURS', val: timeLeft.hours },
            { label: 'MINUTES', val: timeLeft.minutes },
            { label: 'SECONDS', val: timeLeft.seconds },
          ].map((item, idx) => (
            <div key={idx} className="bg-slate-950/80 border border-slate-800 rounded-xl p-3 text-center">
              <div className="text-2xl sm:text-3xl font-extrabold text-indigo-400 font-mono">
                {String(item.val).padStart(2, '0')}
              </div>
              <div className="text-[10px] font-semibold text-slate-500 mt-1">{item.label}</div>
            </div>
          ))}
        </div>
      );
    }

    function getDaysUntil(dateStr) {
      const diff = new Date(dateStr) - new Date();
      return Math.max(0, Math.ceil(diff / (1000 * 60 * 60 * 24)));
    }

    // --- EXAMS VIEW ---
    function ExamsView({ exams, topics, onAddExam, onEditExam, onDeleteExam }) {
      return (
        <div className="max-w-7xl mx-auto space-y-6">
          <div className="flex items-center justify-between">
            <div>
              <h2 className="text-xl font-bold text-white">Scheduled Exams</h2>
              <p className="text-xs text-slate-400">Manage exam dates and targets</p>
            </div>
            <button onClick={onAddExam} className="flex items-center space-x-2 bg-indigo-600 hover:bg-indigo-500 text-white px-3.5 py-2 rounded-xl text-xs font-semibold">
              <i data-lucide="plus" className="w-4 h-4"></i>
              <span>Add Exam</span>
            </button>
          </div>

          <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
            {exams.map(exam => {
              const daysLeft = getDaysUntil(exam.date);
              return (
                <div key={exam.id} className="bg-slate-900 border border-slate-800 rounded-2xl p-6 flex flex-col justify-between">
                  <div>
                    <div className="flex items-center justify-between mb-3">
                      <span className="px-2.5 py-1 text-[10px] font-bold rounded-full uppercase" style={{ backgroundColor: `${exam.color || '#6366f1'}20`, color: exam.color || '#6366f1' }}>
                        {exam.subject}
                      </span>
                      <span className="text-[10px] font-semibold px-2 py-0.5 rounded border bg-red-500/10 text-red-400 border-red-500/20">{exam.priority} Priority</span>
                    </div>
                    <h3 className="font-bold text-lg text-white mb-2">{exam.title}</h3>
                    <p className="text-xs text-slate-400 mb-4">Date: <strong className="text-slate-200">{exam.date}</strong> at {exam.time || '09:00'}</p>
                  </div>
                  <div className="pt-4 border-t border-slate-800 flex items-center justify-between">
                    <span className="text-xs font-semibold text-indigo-400">{daysLeft === 0 ? 'Exam Today!' : `${daysLeft} Days Left`}</span>
                    <div className="flex space-x-1">
                      <button onClick={() => onEditExam(exam)} className="p-1.5 text-slate-400 hover:text-slate-200"><i data-lucide="edit" className="w-4 h-4"></i></button>
                      <button onClick={() => onDeleteExam(exam.id)} className="p-1.5 text-slate-400 hover:text-red-400"><i data-lucide="trash-2" className="w-4 h-4"></i></button>
                    </div>
                  </div>
                </div>
              );
            })}
          </div>
        </div>
      );
    }

    // --- SYLLABUS VIEW ---
    function SyllabusView({ exams, topics, onAddTopic, onUpdateStatus, onDeleteTopic }) {
      const [selectedExamId, setSelectedExamId] = useState(exams[0]?.id || '');
      const [newTopicInput, setNewTopicInput] = useState('');

      const examTopics = topics.filter(t => t.examId === selectedExamId);

      const handleCreate = (e) => {
        e.preventDefault();
        if (!newTopicInput.trim() || !selectedExamId) return;
        onAddTopic(newTopicInput, selectedExamId);
        setNewTopicInput('');
      };

      return (
        <div className="max-w-7xl mx-auto space-y-6">
          <h2 className="text-xl font-bold text-white">Syllabus Breakdown</h2>
          <div className="flex space-x-2 border-b border-slate-800 pb-3">
            {exams.map(e => (
              <button key={e.id} onClick={() => setSelectedExamId(e.id)} className={`px-4 py-2 rounded-xl text-xs font-semibold ${selectedExamId === e.id ? 'bg-indigo-600 text-white' : 'bg-slate-900 text-slate-400'}`}>
                {e.subject}
              </button>
            ))}
          </div>

          <form onSubmit={handleCreate} className="flex gap-2">
            <input type="text" placeholder="Add new topic..." value={newTopicInput} onChange={(e) => setNewTopicInput(e.target.value)} className="flex-1 bg-slate-900 border border-slate-800 rounded-xl px-4 py-2.5 text-sm text-slate-100" />
            <button type="submit" className="bg-indigo-600 hover:bg-indigo-500 text-white px-5 py-2.5 rounded-xl text-xs font-semibold">Add</button>
          </form>

          <div className="space-y-3">
            {examTopics.map(topic => (
              <div key={topic.id} className="bg-slate-900 border border-slate-800 rounded-xl p-4 flex items-center justify-between">
                <span className="text-sm text-slate-100">{topic.name}</span>
                <div className="flex items-center space-x-3">
                  <select value={topic.status} onChange={(e) => onUpdateStatus(topic.id, e.target.value)} className="text-xs bg-slate-950 border border-slate-700 px-3 py-1.5 rounded-lg text-slate-200">
                    <option value="Not Started">Not Started</option>
                    <option value="In Progress">In Progress</option>
                    <option value="Reviewed">Reviewed</option>
                    <option value="Mastered">Mastered</option>
                  </select>
                  <button onClick={() => onDeleteTopic(topic.id)} className="text-slate-500 hover:text-red-400"><i data-lucide="trash-2" className="w-4 h-4"></i></button>
                </div>
              </div>
            ))}
          </div>
        </div>
      );
    }

    // --- POMODORO TIMER VIEW ---
    function PomodoroTimerView({ exams, topics, onLogSession, soundEnabled }) {
      const [secondsLeft, setSecondsLeft] = useState(25 * 60);
      const [isRunning, setIsRunning] = useState(false);
      const [selectedExamId, setSelectedExamId] = useState(exams[0]?.id || '');

      useEffect(() => {
        let timer = null;
        if (isRunning) {
          timer = setInterval(() => {
            setSecondsLeft(prev => {
              if (prev <= 1) {
                clearInterval(timer);
                setIsRunning(false);
                if (soundEnabled) playChimeSound();
                onLogSession({ examId: selectedExamId, durationMinutes: 25 });
                return 0;
              }
              return prev - 1;
            });
          }, 1000);
        }
        return () => clearInterval(timer);
      }, [isRunning, selectedExamId, soundEnabled, onLogSession]);

      const formatTime = (secs) => {
        const m = Math.floor(secs / 60);
        const s = secs % 60;
        return `${String(m).padStart(2, '0')}:${String(s).padStart(2, '0')}`;
      };

      return (
        <div className="max-w-2xl mx-auto space-y-8 text-center">
          <h2 className="text-xl font-bold text-white">Focus Pomodoro Timer</h2>
          <div className="bg-slate-900 border border-slate-800 rounded-3xl p-10 shadow-2xl">
            <div className="text-7xl font-mono text-white font-black my-8">{formatTime(secondsLeft)}</div>
            <div className="flex justify-center space-x-4">
              <button onClick={() => setIsRunning(!isRunning)} className="bg-indigo-600 hover:bg-indigo-500 text-white px-8 py-3.5 rounded-2xl font-bold">
                {isRunning ? 'Pause' : 'Start Focus'}
              </button>
              <button onClick={() => { setIsRunning(false); setSecondsLeft(25 * 60); }} className="bg-slate-800 text-slate-300 p-3.5 rounded-2xl">
                <i data-lucide="rotate-ccw" className="w-5 h-5"></i>
              </button>
            </div>
          </div>
        </div>
      );
    }

    // --- ANALYTICS VIEW ---
    function AnalyticsView({ exams, studySessions }) {
      return (
        <div className="max-w-4xl mx-auto space-y-6">
          <h2 className="text-xl font-bold text-white">Study Analytics</h2>
          <div className="bg-slate-900 border border-slate-800 rounded-2xl p-6">
            <h3 className="text-sm font-semibold text-slate-300 mb-4">Total Sessions Completed</h3>
            <p className="text-3xl font-bold text-indigo-400">{studySessions.length} Study Cycles</p>
          </div>
        </div>
      );
    }

    // --- EXAM MODAL ---
    function ExamModal({ exam, onClose, onSave }) {
      const [subject, setSubject] = useState(exam?.subject || '');
      const [title, setTitle] = useState(exam?.title || '');
      const [date, setDate] = useState(exam?.date || new Date().toISOString().split('T')[0]);

      const handleSubmit = (e) => {
        e.preventDefault();
        onSave({ id: exam?.id, subject, title, date, priority: 'High', color: '#6366f1' });
      };

      return (
        <div className="fixed inset-0 bg-slate-950/80 backdrop-blur-sm flex items-center justify-center p-4 z-50">
          <div className="bg-slate-900 border border-slate-800 rounded-2xl p-6 w-full max-w-md space-y-4">
            <h3 className="text-lg font-bold text-white">{exam ? 'Edit Exam' : 'Add Exam'}</h3>
            <form onSubmit={handleSubmit} className="space-y-3">
              <input type="text" required placeholder="Subject Name" value={subject} onChange={(e) => setSubject(e.target.value)} className="w-full bg-slate-950 border border-slate-800 rounded-xl px-3.5 py-2 text-sm text-slate-200" />
              <input type="text" required placeholder="Exam Title" value={title} onChange={(e) => setTitle(e.target.value)} className="w-full bg-slate-950 border border-slate-800 rounded-xl px-3.5 py-2 text-sm text-slate-200" />
              <input type="date" required value={date} onChange={(e) => setDate(e.target.value)} className="w-full bg-slate-950 border border-slate-800 rounded-xl px-3.5 py-2 text-sm text-slate-200" />
              <div className="flex justify-end space-x-2 pt-2">
                <button type="button" onClick={onClose} className="px-4 py-2 text-xs text-slate-400">Cancel</button>
                <button type="submit" className="px-4 py-2 bg-indigo-600 text-white text-xs font-semibold rounded-xl">Save</button>
              </div>
            </form>
          </div>
        </div>
      );
    }

    const root = ReactDOM.createRoot(document.getElementById('root'));
    root.render(<App />);
  </script>
</body>
</html>