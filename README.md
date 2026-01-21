<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>磨耳朵英语 | MoErDuo English</title>
    
    <!-- 1. 引入 Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    
    <!-- 2. 引入 Babel 用于解析 JSX -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>

    <!-- 3. 定义 Import Map -->
    <script type="importmap">
    {
        "imports": {
            "react": "https://esm.sh/react@18.2.0",
            "react-dom/client": "https://esm.sh/react-dom@18.2.0/client",
            "lucide-react": "https://esm.sh/lucide-react@0.292.0"
        }
    }
    </script>

    <style>
        .animate-spin-slow {
            animation: spin 3s linear infinite;
        }
        @keyframes spin {
            from { transform: rotate(0deg); }
            to { transform: rotate(-360deg); }
        }
        ::selection {
            background-color: #bfdbfe;
        }
        .custom-scrollbar::-webkit-scrollbar {
            width: 6px;
        }
        .custom-scrollbar::-webkit-scrollbar-track {
            background: #f1f5f9;
        }
        .custom-scrollbar::-webkit-scrollbar-thumb {
            background: #cbd5e1;
            border-radius: 3px;
        }
        .custom-scrollbar::-webkit-scrollbar-thumb:hover {
            background: #94a3b8;
        }
    </style>
</head>
<body class="bg-slate-50 text-slate-800 font-sans">
    <div id="root"></div>

    <!-- 应用逻辑 -->
    <script type="text/babel" data-type="module">
        import React, { useState, useEffect, useRef, useCallback } from 'react';
        import { createRoot } from 'react-dom/client';
        import { 
          Play, Pause, RotateCcw, Upload, FileText, Mic, 
          Download, CheckCircle, AlertCircle, Volume2, 
          Languages, X, ChevronDown, User, Sparkles, RefreshCw,
          ThumbsUp, History, Clock, Trash2, FileDown, FileAudio, Music, Square, ImagePlus, ScanText, Loader2
        } from 'lucide-react';

        // --- 音频处理工具函数 ---
        
        // 1. Base64 转 Int16Array (PCM数据)
        const base64ToInt16Array = (base64) => {
            const binaryString = window.atob(base64);
            const len = binaryString.length;
            const bytes = new Uint8Array(len);
            for (let i = 0; i < len; i++) {
                bytes[i] = binaryString.charCodeAt(i);
            }
            // 转换为 16-bit PCM
            return new Int16Array(bytes.buffer);
        };

        // 2. 创建 WAV 文件头并合并数据
        const createWavFile = (pcmData, sampleRate = 24000) => {
            const numChannels = 1;
            const bitsPerSample = 16;
            const byteRate = sampleRate * numChannels * (bitsPerSample / 8);
            const blockAlign = numChannels * (bitsPerSample / 8);
            const dataSize = pcmData.byteLength;
            const buffer = new ArrayBuffer(44 + dataSize);
            const view = new DataView(buffer);

            const writeString = (offset, string) => {
                for (let i = 0; i < string.length; i++) {
                    view.setUint8(offset + i, string.charCodeAt(i));
                }
            };

            writeString(0, 'RIFF');
            view.setUint32(4, 36 + dataSize, true);
            writeString(8, 'WAVE');
            writeString(12, 'fmt ');
            view.setUint32(16, 16, true);
            view.setUint16(20, 1, true); // PCM
            view.setUint16(22, numChannels, true);
            view.setUint32(24, sampleRate, true);
            view.setUint32(28, byteRate, true);
            view.setUint16(32, blockAlign, true);
            view.setUint16(34, bitsPerSample, true);
            writeString(36, 'data');
            view.setUint32(40, dataSize, true);

            // 写入 PCM 数据
            const pcmBytes = new Uint8Array(pcmData.buffer);
            const wavBytes = new Uint8Array(buffer);
            wavBytes.set(new Uint8Array(buffer.slice(0, 44)), 0);
            wavBytes.set(pcmBytes, 44);

            return new Blob([wavBytes], { type: 'audio/wav' });
        };

        // 3. 拼接音频 (循环 N 次)
        const loopPcmData = (originalPcm, loopCount, silenceMs = 800, sampleRate = 24000) => {
            const silenceSamplesCount = Math.floor((silenceMs / 1000) * sampleRate);
            const singleLoopLength = originalPcm.length + silenceSamplesCount;
            const totalLength = singleLoopLength * loopCount;
            
            // 安全检查：如果音频过大 (超过 ~200MB)，限制循环次数以防崩溃
            if (totalLength * 2 > 200 * 1024 * 1024) {
                console.warn("Audio too large, truncating loop count.");
                loopCount = Math.floor((100 * 1024 * 1024) / (singleLoopLength * 2));
                alert(`文本过长，为防止浏览器崩溃，自动调整为循环 ${loopCount} 遍。`);
                return loopPcmData(originalPcm, loopCount, silenceMs, sampleRate);
            }

            const result = new Int16Array(totalLength);
            
            for (let i = 0; i < loopCount; i++) {
                const offset = i * singleLoopLength;
                result.set(originalPcm, offset);
                // 剩余部分默认为0 (静音)
            }
            return result;
        };

        const App = () => {
          // --- 状态管理 ---
          const [text, setText] = useState("London is the capital and largest city of England and the United Kingdom. It stands on the River Thames in south-east England at the head of a 50-mile estuary down to the North Sea.");
          const [translation, setTranslation] = useState("伦敦是英格兰和联合王国的首都及最大城市。它位于英格兰东南部的泰晤士河畔，处于通往北海的50英里长河口的顶端。（点击此处可编辑翻译）");
          const [isProcessingImage, setIsProcessingImage] = useState(false);
          const [isTranslating, setIsTranslating] = useState(false);
          const [voices, setVoices] = useState([]);
          const [selectedVoice, setSelectedVoice] = useState(null);
          const [playbackSpeed, setPlaybackSpeed] = useState(1.0);
          const [loopCount, setLoopCount] = useState(1);
          const [currentLoop, setCurrentLoop] = useState(0);
          const [isPlaying, setIsPlaying] = useState(false);
          const [highlightIndex, setHighlightIndex] = useState(-1);
          
          // 历史记录与下载
          const [history, setHistory] = useState([]);
          const [downloads, setDownloads] = useState([]);
          const [generatingId, setGeneratingId] = useState(null); 

          // 跟读相关
          const [isRecording, setIsRecording] = useState(false);
          const [spokenText, setSpokenText] = useState("");
          const [diffResult, setDiffResult] = useState(null); 

          const synthesisRef = useRef(window.speechSynthesis);
          const utteranceRef = useRef(null);
          const recognitionRef = useRef(null);
          const isStoppedRef = useRef(false);

          // --- 初始化 ---
          useEffect(() => {
            const loadVoices = () => {
              const allVoices = synthesisRef.current.getVoices();
              const britishVoices = allVoices.filter(v => v.lang === 'en-GB');
              const availableVoices = britishVoices.length > 0 ? britishVoices : allVoices.filter(v => v.lang.startsWith('en'));
              setVoices(availableVoices);
              if (availableVoices.length > 0) {
                const preferred = availableVoices.find(v => v.name.includes('Female') || v.name.includes('Hazel') || v.name.includes('Google UK English Female')) || availableVoices[0];
                setSelectedVoice(preferred);
              }
            };
            loadVoices();
            if (window.speechSynthesis.onvoiceschanged !== undefined) {
              window.speechSynthesis.onvoiceschanged = loadVoices;
            }

            if (!window.Tesseract) {
                const script = document.createElement('script');
                script.src = "https://cdn.jsdelivr.net/npm/tesseract.js@5/dist/tesseract.min.js";
                script.async = true;
                document.body.appendChild(script);
            }

            const savedHistory = localStorage.getItem('moerduo_history');
            if (savedHistory) {
                try {
                    setHistory(JSON.parse(savedHistory));
                } catch (e) {
                    console.error("无法解析历史记录", e);
                }
            }

            return () => {
              if (synthesisRef.current) synthesisRef.current.cancel();
            };
          }, []);

          // --- 历史记录管理 ---
          const saveToHistory = (currentText, currentTrans) => {
            if (!currentText || currentText.length < 5) return;

            setHistory(prev => {
                if (prev.length > 0 && prev[0].text === currentText) return prev;
                const newRecord = {
                    id: Date.now(),
                    text: currentText,
                    translation: currentTrans,
                    date: new Date().toLocaleString()
                };
                const newHistory = [newRecord, ...prev].slice(0, 10);
                localStorage.setItem('moerduo_history', JSON.stringify(newHistory));
                return newHistory;
            });
          };

          const loadHistoryItem = (item) => {
             if (text && text !== item.text && !window.confirm("确定要加载这条记录吗？当前编辑框的内容将被覆盖。")) return;
             setText(item.text);
             setTranslation(item.translation);
             window.scrollTo({ top: 0, behavior: 'smooth' });
          };

          const deleteHistoryItem = (e, id) => {
              e.stopPropagation();
              const newHistory = history.filter(item => item.id !== id);
              setHistory(newHistory);
              localStorage.setItem('moerduo_history', JSON.stringify(newHistory));
          };

          // --- 下载资源管理 ---
          const addToDownloads = (txt, trans, count) => {
              if (count !== 50) return;

              setDownloads(prev => {
                  const newDownload = {
                      id: Date.now(),
                      title: `磨耳朵资源 (50遍)`,
                      count: count,
                      text: txt,
                      trans: trans,
                      time: new Date().toLocaleTimeString()
                  };
                  return [newDownload, ...prev].slice(0, 5);
              });
          };

          const handleDownloadScript = (item) => {
              let content = `磨耳朵英语 - 学习讲义\n生成时间: ${new Date().toLocaleString()}\n-------------------------\n\n原文:\n${item.text}\n\n翻译:\n${item.trans}\n\n`;
              content += `-------------------------\n以下是 ${item.count} 遍循环练习文本 (Study Script):\n\n`;
              for(let i=1; i<=item.count; i++) {
                  content += `[第 ${i} 遍]\n${item.text}\n\n`;
              }
              
              const blob = new Blob([content], { type: 'text/plain;charset=utf-8' });
              const url = URL.createObjectURL(blob);
              const a = document.createElement('a');
              a.href = url;
              a.download = `MoErDuo_Script_${item.count}x.txt`;
              document.body.appendChild(a);
              a.click();
              document.body.removeChild(a);
              URL.revokeObjectURL(url);
          };

          // --- 核心：云端生成 + 本地拼接音频 ---
          const handleGenerateAudio = async (item) => {
              if (generatingId) return; 
              setGeneratingId(item.id);

              try {
                  const apiKey = ""; // API Key 自动注入
                  const url = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-tts:generateContent?key=${apiKey}`;
                  
                  let geminiVoice = "Kore"; 
                  if (selectedVoice && selectedVoice.name.toLowerCase().includes("male") && !selectedVoice.name.toLowerCase().includes("female")) {
                      geminiVoice = "Puck";
                  }

                  let response;
                  let attempt = 0;
                  const maxRetries = 3;
                  
                  while (attempt < maxRetries) {
                      try {
                          response = await fetch(url, {
                              method: 'POST',
                              headers: { 'Content-Type': 'application/json' },
                              body: JSON.stringify({
                                  contents: [{ parts: [{ text: item.text }] }],
                                  generationConfig: {
                                      responseModalities: ["AUDIO"],
                                      speechConfig: {
                                          voiceConfig: {
                                              prebuiltVoiceConfig: { voiceName: geminiVoice }
                                          }
                                      }
                                  }
                              })
                          });
                          
                          if (response.status === 429 || response.status >= 500) {
                              throw new Error(`Server busy (${response.status})`);
                          }
                          break;
                      } catch (err) {
                          attempt++;
                          console.warn(`第 ${attempt} 次尝试失败:`, err);
                          if (attempt >= maxRetries) break;
                          await new Promise(resolve => setTimeout(resolve, Math.pow(2, attempt - 1) * 1000));
                      }
                  }

                  if (!response || !response.ok) {
                      let errMsg = "语音生成服务繁忙，请稍后再试";
                      try {
                          const errData = await response.json();
                          if (errData.error && errData.error.message) {
                              errMsg = `API错误: ${errData.error.message}`;
                          }
                      } catch (e) {}
                      throw new Error(errMsg);
                  }

                  const data = await response.json();
                  const inlineData = data.candidates?.[0]?.content?.parts?.[0]?.inlineData;
                  
                  if (!inlineData || !inlineData.data) throw new Error("未获取到音频数据");

                  const pcmData = base64ToInt16Array(inlineData.data);
                  const loopedPcmData = loopPcmData(pcmData, item.count); 
                  const wavBlob = createWavFile(loopedPcmData, 24000);

                  const downloadUrl = URL.createObjectURL(wavBlob);
                  const a = document.createElement('a');
                  a.href = downloadUrl;
                  a.download = `MoErDuo_Audio_Loop${item.count}x_${Date.now()}.wav`;
                  document.body.appendChild(a);
                  a.click();
                  document.body.removeChild(a);
                  URL.revokeObjectURL(downloadUrl);

                  alert("音频文件生成成功！已开始下载。");

              } catch (err) {
                  console.error("音频生成失败", err);
                  alert(`生成失败: ${err.message || "请检查网络连接"}`);
              } finally {
                  setGeneratingId(null);
              }
          };

          // --- 自动翻译监听 ---
          useEffect(() => {
            const timer = setTimeout(async () => {
                if (!text.trim()) {
                    setTranslation("");
                    return;
                }
                
                setIsTranslating(true);
                try {
                    const limit = 1000; 
                    const queryText = text.length > limit ? text.substring(0, limit) : text;
                    const res = await fetch(`https://api.mymemory.translated.net/get?q=${encodeURIComponent(queryText)}&langpair=en|zh-CN`);
                    if (!res.ok) throw new Error('Network response was not ok');
                    const data = await res.json();
                    
                    if (data.responseStatus === 200) {
                        const resultText = text.length > limit 
                            ? data.responseData.translatedText + `... (长文本仅翻译前${limit}字)` 
                            : data.responseData.translatedText;
                        setTranslation(resultText);
                    } else {
                         console.warn("Translation API warning:", data.responseDetails);
                    }
                } catch (error) {
                    console.warn("自动翻译请求受限或网络问题，已忽略:", error);
                } finally {
                    setIsTranslating(false);
                }
            }, 1000);

            return () => clearTimeout(timer);
          }, [text]);

          // --- 逻辑功能 ---

          // 1. OCR 图片识别
          const handleImageUpload = async (e) => {
            const file = e.target.files[0];
            if (!file) return;
            if (!window.Tesseract) { alert("OCR 组件正在加载中，请稍后再试..."); return; }

            setIsProcessingImage(true);
            try {
              const { data: { text: recognizedText } } = await window.Tesseract.recognize(file, 'eng');
              const cleanText = recognizedText.replace(/\s+/g, ' ').trim();
              setText(cleanText);
              saveToHistory(cleanText, "（等待翻译...）"); 
            } catch (err) {
              alert("图片识别失败，请重试或检查图片清晰度。");
            } finally {
              setIsProcessingImage(false);
            }
          };

          // 2. 语音合成 (TTS) 控制 (本地播放)
          const cancelSpeech = () => {
            if (synthesisRef.current) synthesisRef.current.cancel();
            setIsPlaying(false);
            setHighlightIndex(-1);
            setCurrentLoop(0);
          };

          const speak = (targetText, loopTarget = 1) => {
            if (!targetText) return;

            isStoppedRef.current = false;
            saveToHistory(targetText, translation);
            addToDownloads(targetText, translation, 50); // 始终生成 50 遍下载项
            
            synthesisRef.current.cancel();

            const playNext = (remainingLoops, currentLoopIndex) => {
                if (isStoppedRef.current || remainingLoops <= 0) {
                    setIsPlaying(false);
                    setHighlightIndex(-1);
                    setCurrentLoop(0);
                    return;
                }

                const u = new SpeechSynthesisUtterance(targetText);
                utteranceRef.current = u; 
                u.voice = selectedVoice;
                u.rate = playbackSpeed;
                
                u.onboundary = (event) => { 
                    if (event.name === 'word') setHighlightIndex(event.charIndex); 
                };
                
                u.onend = () => {
                    setCurrentLoop(currentLoopIndex + 1);
                    setTimeout(() => {
                        if (isStoppedRef.current) return;
                        playNext(remainingLoops - 1, currentLoopIndex + 1);
                    }, 500);
                };

                u.onerror = () => setIsPlaying(false);
                synthesisRef.current.speak(u);
            };

            setIsPlaying(true);
            setCurrentLoop(0);
            playNext(loopTarget, 0);
          };

          const togglePlay = () => {
            if (isPlaying) {
              isStoppedRef.current = true;
              if (synthesisRef.current) synthesisRef.current.cancel();
              setIsPlaying(false);
              setHighlightIndex(-1);
              setCurrentLoop(0);
            } else {
              setCurrentLoop(0);
              speak(text, loopCount);
            }
          };

          // 3. 跟读与纠错 (STT)
          const startRecording = async () => {
            const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
            if (!SpeechRecognition) {
              alert("您的浏览器不支持语音识别，建议使用 Chrome 或 Edge 桌面版。");
              return;
            }

            try {
                await navigator.mediaDevices.getUserMedia({ audio: true });
            } catch (err) {
                console.error("无法获取麦克风权限", err);
            }

            recognitionRef.current = new SpeechRecognition();
            recognitionRef.current.lang = 'en-US'; 
            recognitionRef.current.continuous = true;
            recognitionRef.current.interimResults = true;

            recognitionRef.current.onresult = (event) => {
              let transcript = '';
              for (let i = event.resultIndex; i < event.results.length; ++i) {
                transcript += event.results[i][0].transcript;
              }
              setSpokenText(transcript);
            };

            recognitionRef.current.onerror = (event) => {
              console.error("语音识别错误", event);
              setIsRecording(false);
            };

            try {
                recognitionRef.current.start();
                setIsRecording(true);
                setDiffResult(null); 
            } catch (error) {
                alert("无法启动语音识别，请刷新页面重试。");
                setIsRecording(false);
            }
          };

          const stopRecording = () => {
            if (recognitionRef.current) {
              try { recognitionRef.current.stop(); } catch(e) {}
            }
            setIsRecording(false);
            analyzePronunciationAdvanced();
          };

          const analyzePronunciationAdvanced = () => {
            const originalTokens = text.trim().split(/\s+/).map(t => ({
              text: t,
              clean: t.toLowerCase().replace(/[^\w']/g, ''), 
              status: 'missing' 
            }));

            const spokenTokens = spokenText.trim().split(/\s+/).map(t => 
              t.toLowerCase().replace(/[^\w']/g, '')
            ).filter(t => t.length > 0);

            const n = originalTokens.length;
            const m = spokenTokens.length;
            const dp = Array(n + 1).fill(null).map(() => Array(m + 1).fill(0));

            for (let i = 1; i <= n; i++) {
              for (let j = 1; j <= m; j++) {
                if (originalTokens[i - 1].clean === spokenTokens[j - 1]) {
                  dp[i][j] = dp[i - 1][j - 1] + 1;
                } else {
                  dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
                }
              }
            }

            let i = n, j = m;
            while (i > 0 && j > 0) {
              if (originalTokens[i - 1].clean === spokenTokens[j - 1]) {
                originalTokens[i - 1].status = 'correct';
                i--;
                j--;
              } else if (dp[i - 1][j] > dp[i][j - 1]) {
                i--;
              } else {
                j--;
              }
            }

            const sentences = [];
            let currentSentence = [];
            
            originalTokens.forEach(token => {
              currentSentence.push(token);
              if (/[.!?]["']?$/.test(token.text)) {
                sentences.push(currentSentence);
                currentSentence = [];
              }
            });
            if (currentSentence.length > 0) sentences.push(currentSentence);

            const result = sentences.map(s => ({
              words: s,
              hasError: s.some(w => w.status === 'missing')
            }));

            setDiffResult(result);
          };

          const renderHighlightedText = () => {
            if (highlightIndex === -1) return text;
            const before = text.substring(0, highlightIndex);
            const remainder = text.substring(highlightIndex);
            const wordMatch = remainder.match(/^(\S+)/); 
            const word = wordMatch ? wordMatch[0] : "";
            const after = remainder.substring(word.length);

            return (
              <span>
                {before}
                <span className="bg-yellow-300 text-black px-1 rounded transition-all duration-75 border-b-2 border-orange-500">
                    {word}
                </span>
                {after}
              </span>
            );
          };

          return (
            <div className="min-h-screen pb-12">
              {/* 头部 */}
              <header className="bg-white shadow-sm sticky top-0 z-20">
                <div className="max-w-4xl mx-auto px-4 py-3 flex items-center justify-between">
                  <div className="flex items-center gap-2 text-blue-600">
                    <div className="p-2 bg-blue-100 rounded-lg"><Volume2 size={24} /></div>
                    <h1 className="text-xl font-bold tracking-tight">磨耳朵英语 <span className="text-xs font-normal text-slate-500 block sm:inline">| MoErDuo English</span></h1>
                  </div>
                  <div className="text-xs text-slate-400 hidden sm:block">正版英式发音 · 智能循环 · 跟读纠错</div>
                </div>
              </header>

              <main className="max-w-4xl mx-auto px-4 py-6 space-y-6">
                
                {/* 1. 输入区域 - 左右分栏 */}
                <section className="bg-white rounded-2xl shadow-sm border border-slate-100 overflow-hidden relative">
                  <div className="p-4 bg-slate-50 border-b border-slate-100 flex justify-between items-center">
                    <h2 className="font-semibold text-slate-700 flex items-center gap-2"><FileText size={18} /> 学习内容</h2>
                    <div className="flex gap-2">
                       <button type="button" onClick={() => { setText(""); setTranslation(""); }} className="px-3 py-1.5 text-xs text-slate-400 hover:text-red-500 transition-colors">清空</button>
                    </div>
                  </div>
                  
                  <div className="grid md:grid-cols-2 divide-y md:divide-y-0 md:divide-x divide-slate-100">
                    <div className="p-4 relative bg-white">
                        <div className="mb-2 flex justify-between items-center">
                          <span className="text-xs font-bold text-blue-600 uppercase tracking-wider">English (英文文本)</span>
                          <span className="text-xs text-slate-400">{text.length} 字符</span>
                        </div>
                        <textarea 
                            value={text} 
                            onChange={(e) => setText(e.target.value)} 
                            className="w-full h-48 p-2 resize-y focus:outline-none focus:ring-2 focus:ring-blue-100 rounded-md text-lg leading-relaxed text-slate-700 font-medium bg-transparent" 
                            placeholder="在这里输入或粘贴英文内容..." 
                        />
                    </div>

                    <div className="p-4 bg-slate-50/50 flex flex-col justify-center relative">
                        <div className="mb-2 flex justify-between items-center w-full">
                            <span className="text-xs font-bold text-slate-500 uppercase tracking-wider flex items-center gap-2">
                                {text ? (
                                    <><RefreshCw size={14} className={isTranslating ? "animate-spin" : ""} /> 中文翻译 (自动)</>
                                ) : (
                                    <><ScanText size={14} /> 上传图片识别 (OCR)</>
                                )}
                            </span>
                            {text && (
                                <button 
                                    onClick={() => setText("")} 
                                    className="text-xs text-blue-500 hover:underline flex items-center gap-1"
                                    title="清空并重新上传"
                                >
                                    <ImagePlus size={12}/> 重新上传
                                </button>
                            )}
                        </div>
                        
                        {!text ? (
                            <label className="flex-1 flex flex-col items-center justify-center gap-3 p-6 bg-white border-2 border-dashed border-slate-200 hover:border-blue-400 rounded-xl cursor-pointer transition-all group hover:bg-blue-50/30">
                                <div className="p-4 bg-slate-100 group-hover:bg-blue-100 rounded-full transition-colors">
                                    {isProcessingImage ? (
                                        <RefreshCw size={32} className="text-blue-500 animate-spin" />
                                    ) : (
                                        <ImagePlus size={32} className="text-slate-400 group-hover:text-blue-500 transition-colors" />
                                    )}
                                </div>
                                <div className="text-center">
                                    <p className="text-sm font-bold text-slate-600 group-hover:text-blue-600 transition-colors">
                                        {isProcessingImage ? '正在识别文字...' : '点击上传图片'}
                                    </p>
                                    <p className="text-xs text-slate-400 mt-1">支持自动提取英文文本并填入左侧</p>
                                </div>
                                <input type="file" accept="image/*" className="hidden" onChange={handleImageUpload} disabled={isProcessingImage} />
                            </label>
                        ) : (
                            <textarea 
                                value={translation} 
                                onChange={(e) => setTranslation(e.target.value)} 
                                className="w-full h-48 p-2 resize-y bg-transparent focus:outline-none focus:ring-2 focus:ring-slate-200 rounded-md text-base leading-relaxed text-slate-600" 
                                placeholder="翻译生成中..." 
                            />
                        )}
                    </div>
                  </div>
                </section>

                {/* 2. 播放控制与显示区 */}
                <section className="bg-white rounded-2xl shadow-lg border border-blue-100 overflow-hidden relative">
                  <div className="absolute top-0 left-0 w-full h-1 bg-slate-100">
                    {isPlaying && (<div className="h-full bg-blue-500 animate-pulse" style={{ width: '100%' }}></div>)}
                  </div>
                  <div className="p-6">
                    <div className="flex flex-wrap items-center justify-between gap-4 mb-6">
                       <div className="flex flex-wrap items-center gap-3">
                          <div className="relative group">
                            <div className="flex items-center gap-2 px-3 py-2 bg-slate-100 rounded-lg text-sm font-medium text-slate-700 min-w-[180px] cursor-pointer hover:bg-slate-200 transition-colors">
                                <User size={16} className="text-blue-500"/>
                                <span className="truncate max-w-[140px]">{selectedVoice ? selectedVoice.name.replace('Microsoft', '').replace('Google', '').trim() : '加载声音中...'}</span>
                                <ChevronDown size={14} className="ml-auto opacity-50"/>
                            </div>
                            <select className="absolute inset-0 opacity-0 cursor-pointer" onChange={(e) => { const v = voices.find(voice => voice.name === e.target.value); setSelectedVoice(v); }}>
                                {voices.map(v => (<option key={v.name} value={v.name}>{v.name} ({v.lang})</option>))}
                            </select>
                          </div>
                          <div className="flex items-center bg-slate-100 rounded-lg p-1">
                            {[0.8, 1.0, 1.2, 1.5].map(speed => (<button key={speed} type="button" onClick={() => setPlaybackSpeed(speed)} className={`px-2 py-1 text-xs font-bold rounded-md transition-all ${playbackSpeed === speed ? 'bg-white shadow text-blue-600' : 'text-slate-500 hover:text-slate-700'}`}>{speed}x</button>))}
                          </div>
                       </div>
                       <div className="flex items-center gap-2">
                           <span className="text-xs text-slate-500 font-medium uppercase">播放次数</span>
                           <div className="flex bg-blue-50 rounded-lg p-1">
                               {[1, 5, 10, 50].map(count => (<button key={count} type="button" onClick={() => setLoopCount(count)} className={`px-3 py-1 text-xs font-bold rounded-md transition-all ${loopCount === count ? 'bg-blue-600 text-white shadow-md' : 'text-blue-600/60 hover:bg-blue-100'}`}>{count}</button>))}
                           </div>
                       </div>
                    </div>

                    <div className="bg-slate-50 rounded-xl p-6 md:p-8 min-h-[200px] flex flex-col items-center justify-center text-center border border-slate-200/60 relative">
                        {isPlaying && (<div className="mb-4 inline-flex items-center gap-2 px-3 py-1 bg-green-100 text-green-700 rounded-full text-xs font-bold"><RotateCcw size={12} className={isPlaying ? "animate-spin-slow" : ""} /> 正在播放第 {currentLoop + 1} / {loopCount} 遍</div>)}
                        
                        <div className="text-2xl md:text-3xl font-serif text-slate-800 leading-relaxed max-w-3xl">{isPlaying ? renderHighlightedText() : text}</div>
                        <div className="mt-6 text-slate-500 text-base max-w-2xl border-t border-slate-200 pt-4">{isTranslating ? (<span className="flex items-center justify-center gap-2 text-slate-400"><RefreshCw size={14} className="animate-spin"/> 正在同步翻译...</span>) : translation}</div>
                    </div>

                    <div className="mt-8 flex justify-center">
                        <button type="button" onClick={togglePlay} className={`flex items-center gap-3 px-8 py-4 rounded-full text-lg font-bold shadow-xl transition-all transform hover:scale-105 active:scale-95 ${isPlaying ? 'bg-red-500 hover:bg-red-600 text-white shadow-red-200' : 'bg-blue-600 hover:bg-blue-700 text-white shadow-blue-200'}`}>
                            {isPlaying ? <Pause fill="currentColor" /> : <Play fill="currentColor" />} {isPlaying ? '停止播放' : '开始磨耳朵'}
                        </button>
                    </div>
                  </div>
                </section>

                {/* 3. 跟读与纠错区 */}
                <section className="bg-white rounded-2xl shadow-lg border border-slate-100 overflow-hidden">
                    <div className="p-4 bg-slate-50 border-b border-slate-100 flex items-center gap-2">
                        <Mic size={18} className="text-slate-700"/>
                        <h2 className="font-semibold text-slate-700">跟读挑战 (Shadowing)</h2>
                    </div>
                    
                    <div className="grid md:grid-cols-2 divide-y md:divide-y-0 md:divide-x divide-slate-100">
                        <div className="p-6 flex flex-col">
                            <h3 className="text-xs font-bold text-slate-400 uppercase tracking-wider mb-4">跟读内容 (Target)</h3>
                            <div className="flex-1 text-lg text-slate-700 leading-relaxed mb-6 font-serif">
                                {text}
                            </div>
                            
                            <div className="mt-auto flex justify-center">
                                 <button
                                    type="button"
                                    onClick={isRecording ? stopRecording : startRecording}
                                    className={`w-full py-4 rounded-xl font-bold transition-all flex items-center justify-center gap-3 ${
                                        isRecording 
                                        ? 'bg-red-50 border-2 border-red-500 text-red-600 animate-pulse' 
                                        : 'bg-slate-800 text-white hover:bg-slate-700 shadow-lg'
                                    }`}
                                >
                                    {isRecording ? <Square size={20} /> : <Mic size={20} />}
                                    {isRecording ? '停止录音 (Stop)' : '开始跟读 (Record)'}
                                </button>
                            </div>
                        </div>

                        <div className="p-6 bg-slate-50/50 flex flex-col">
                             <h3 className="text-xs font-bold text-slate-400 uppercase tracking-wider mb-4">评测结果 (Result)</h3>
                             
                             <div className="flex-1 overflow-y-auto max-h-[400px]">
                                {!diffResult && !isRecording ? (
                                    <div className="h-full flex flex-col items-center justify-center text-slate-400 text-sm">
                                        <Sparkles size={32} className="mb-2 opacity-30"/>
                                        <p>请点击左侧按钮开始跟读</p>
                                        <p className="text-xs mt-1">读完后系统将自动标出错误发音</p>
                                    </div>
                                ) : isRecording ? (
                                    <div className="text-slate-500 animate-pulse">
                                        正在聆听您的发音...
                                        <div className="mt-4 p-4 bg-white rounded-lg border border-slate-100 shadow-sm text-sm">
                                            {spokenText || "..."}
                                        </div>
                                    </div>
                                ) : (
                                    <div className="space-y-4">
                                        <div className="flex items-center gap-2 text-sm p-3 rounded-lg bg-white border border-slate-100 shadow-sm">
                                            {diffResult.filter(s => s.hasError).length === 0 ? (
                                                <span className="text-green-600 font-bold flex items-center gap-2"><ThumbsUp size={16}/> 发音完美！</span>
                                            ) : (
                                                <span className="text-orange-500 font-bold flex items-center gap-2"><AlertCircle size={16}/> 发现 {diffResult.filter(s => s.hasError).length} 处需改进</span>
                                            )}
                                        </div>

                                        <div className="text-lg leading-relaxed">
                                            {diffResult.map((sentence, sIdx) => (
                                                <span key={sIdx} className="inline">
                                                    {sentence.words.map((wordObj, wIdx) => {
                                                        const isError = wordObj.status === 'missing';
                                                        return (
                                                            <span 
                                                                key={wIdx} 
                                                                className={`inline-block mr-1.5 px-1 rounded transition-colors ${
                                                                    isError 
                                                                    ? 'bg-red-100 text-red-600 decoration-red-400 underline underline-offset-4 decoration-2' 
                                                                    : 'text-green-700'
                                                                }`}
                                                            >
                                                                {wordObj.text}
                                                            </span>
                                                        );
                                                    })}
                                                    <span className="mr-2"></span> 
                                                </span>
                                            ))}
                                        </div>
                                        
                                        <div className="mt-6 pt-4 border-t border-slate-200">
                                            <span className="text-xs font-bold text-slate-400 block mb-2">原始识别内容:</span>
                                            <p className="text-xs text-slate-500 font-mono bg-white p-2 rounded border border-slate-100">{spokenText}</p>
                                        </div>
                                    </div>
                                )}
                             </div>
                        </div>
                    </div>
                </section>

                {/* 4. 底部区: 历史记录 & 下载 */}
                <div className="grid md:grid-cols-2 gap-6">
                    <section className="bg-white rounded-2xl shadow-sm border border-slate-100 overflow-hidden">
                        <div className="p-4 bg-slate-50 border-b border-slate-100 flex justify-between items-center">
                            <h2 className="font-semibold text-slate-700 flex items-center gap-2"><History size={18} /> 最近学习记录</h2>
                            <span className="text-xs text-slate-400">保留最近10条</span>
                        </div>
                        {history.length === 0 ? (
                            <div className="p-8 text-center text-slate-400 text-sm"><Clock size={32} className="mx-auto mb-2 opacity-30" />暂无记录，播放或上传后自动保存。</div>
                        ) : (
                            <div className="divide-y divide-slate-100 max-h-60 overflow-y-auto custom-scrollbar">
                                {history.map((item) => (
                                    <div key={item.id} onClick={() => loadHistoryItem(item)} className="group flex items-start gap-4 p-4 hover:bg-blue-50 transition-colors cursor-pointer relative">
                                        <div className="flex-1 min-w-0">
                                            <div className="flex items-center gap-2 mb-1"><span className="text-xs font-bold text-slate-400">{item.date}</span></div>
                                            <p className="text-sm font-medium text-slate-700 truncate">{item.text}</p>
                                        </div>
                                        <button onClick={(e) => deleteHistoryItem(e, item.id)} className="p-2 text-slate-300 hover:text-red-500 hover:bg-red-50 rounded-full transition-all opacity-0 group-hover:opacity-100" title="删除"><Trash2 size={16} /></button>
                                    </div>
                                ))}
                            </div>
                        )}
                    </section>

                    <section className="bg-white rounded-2xl shadow-sm border border-slate-100 overflow-hidden">
                        <div className="p-4 bg-slate-50 border-b border-slate-100 flex justify-between items-center">
                            <h2 className="font-semibold text-slate-700 flex items-center gap-2"><FileDown size={18} /> 学习资源下载</h2>
                            <span className="text-xs text-slate-400">仅显示50遍循环资源</span>
                        </div>
                        {downloads.length === 0 ? (
                            <div className="p-8 text-center text-slate-400 text-sm"><FileDown size={32} className="mx-auto mb-2 opacity-30" />点击“开始磨耳朵”后，此处将自动生成50遍循环的资源供下载。</div>
                        ) : (
                            <div className="divide-y divide-slate-100 max-h-60 overflow-y-auto custom-scrollbar">
                                {downloads.map((item) => (
                                    <div key={item.id} className="flex flex-col p-4 hover:bg-slate-50 transition-colors gap-3">
                                        <div>
                                            <div className="flex items-center gap-2">
                                                <span className="text-xs font-bold px-2 py-0.5 bg-blue-100 text-blue-600 rounded-full">{item.count}遍循环</span>
                                                <span className="text-xs text-slate-400">{item.time}</span>
                                            </div>
                                            <p className="text-sm font-medium text-slate-700 mt-1 truncate">{item.text.substring(0, 40)}...</p>
                                        </div>
                                        <div className="flex gap-2">
                                            <button onClick={() => handleDownloadScript(item)} className="flex-1 flex items-center justify-center gap-1 px-3 py-1.5 text-xs font-bold bg-white border border-slate-200 text-slate-600 rounded-lg hover:bg-slate-50 hover:text-blue-600 transition-colors">
                                                <FileText size={14} /> 文本讲义
                                            </button>
                                            
                                            <button 
                                                onClick={() => handleGenerateAudio(item)} 
                                                disabled={generatingId === item.id}
                                                className={`flex-1 flex items-center justify-center gap-1 px-3 py-1.5 text-xs font-bold rounded-lg text-white transition-colors shadow-sm ${
                                                    generatingId === item.id
                                                        ? 'bg-slate-400 cursor-not-allowed' 
                                                        : 'bg-green-600 hover:bg-green-700'
                                                }`}
                                            >
                                                {generatingId === item.id ? (
                                                    <><Loader2 size={12} className="animate-spin" /> 生成中...</>
                                                ) : (
                                                    <><FileAudio size={14} /> 生成50遍音频 (WAV)</>
                                                )}
                                            </button>
                                        </div>
                                    </div>
                                ))}
                            </div>
                        )}
                    </section>
                </div>

              </main>
            </div>
          );
        };

        const root = createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
