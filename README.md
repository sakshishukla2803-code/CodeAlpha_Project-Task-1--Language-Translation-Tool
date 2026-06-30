# CodeAlpha_Project-Task-1--Language-Translation-Tool
Author-Sakshi Shukla


import React, { useState, useEffect, useRef } from "react";
import { motion, AnimatePresence } from "motion/react";
import {
  Languages,
  ArrowLeftRight,
  Copy,
  Check,
  Volume2,
  VolumeX,
  X,
  Sparkles,
  History,
  Trash2,
  AlertCircle,
  HelpCircle,
  Clock,
  ExternalLink
} from "lucide-react";
import { SUPPORTED_LANGUAGES, Language } from "./types";

interface HistoryItem {
  id: string;
  sourceText: string;
  translatedText: string;
  sourceLang: string;
  targetLang: string;
  sourceCode: string;
  targetCode: string;
  timestamp: number;
}

export default function App() {
  // State for texts and languages
  const [sourceText, setSourceText] = useState("");
  const [targetText, setTargetText] = useState("");
  const [sourceLangCode, setSourceLangCode] = useState("auto"); // Default is Auto-Detect
  const [targetLangCode, setTargetLangCode] = useState("es"); // Default is Spanish
  
  // Status states
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [detectedLang, setDetectedLang] = useState<string | null>(null);
  const [detectedCode, setDetectedCode] = useState<string | null>(null);
  const [translationProvider, setTranslationProvider] = useState<string | null>(null);
  
  // Copy and Speech feedback states
  const [sourceCopied, setSourceCopied] = useState(false);
  const [targetCopied, setTargetCopied] = useState(false);
  const [isSpeakingSource, setIsSpeakingSource] = useState(false);
  const [isSpeakingTarget, setIsSpeakingTarget] = useState(false);
  
  // History panel & items
  const [history, setHistory] = useState<HistoryItem[]>([]);
  const [showHistory, setShowHistory] = useState(false);

  // Speech synthesis reference
  const synthRef = useRef<SpeechSynthesis | null>(null);
  const currentUtteranceRef = useRef<SpeechSynthesisUtterance | null>(null);

  // Initialize speech synthesis and load history
  useEffect(() => {
    if (typeof window !== "undefined") {
      synthRef.current = window.speechSynthesis;
      
      // Load history from localStorage
      const savedHistory = localStorage.getItem("translation_history");
      if (savedHistory) {
        try {
          setHistory(JSON.parse(savedHistory));
        } catch (e) {
          console.error("Failed to parse translation history:", e);
        }
      }
    }

    // Cleanup speech on unmount
    return () => {
      if (synthRef.current) {
        synthRef.current.cancel();
      }
    };
  }, []);

  // Sync history to localStorage
  const saveHistoryToStorage = (updatedHistory: HistoryItem[]) => {
    setHistory(updatedHistory);
    localStorage.setItem("translation_history", JSON.stringify(updatedHistory));
  };

  // Find Language objects
  const currentSourceLanguage = sourceLangCode === "auto" 
    ? { name: "Auto-Detect", code: "auto", speechCode: "en-US" }
    : SUPPORTED_LANGUAGES.find((l) => l.code === sourceLangCode) || SUPPORTED_LANGUAGES[0];

  const currentTargetLanguage = SUPPORTED_LANGUAGES.find((l) => l.code === targetLangCode) || SUPPORTED_LANGUAGES[1];

  // Primary translation function
  const handleTranslate = async (textToTranslate: string = sourceText) => {
    if (!textToTranslate.trim()) {
      setTargetText("");
      setError(null);
      return;
    }

    setIsLoading(true);
    setError(null);

    try {
      const response = await fetch("/api/translate", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          text: textToTranslate,
          sourceLang: currentSourceLanguage.name,
          targetLang: currentTargetLanguage.name,
          sourceCode: sourceLangCode,
          targetCode: targetLangCode,
        }),
      });

      const data = await response.json();

      if (!response.ok) {
        throw new Error(data.error || "Translation API failed");
      }

      setTargetText(data.translatedText);
      setTranslationProvider(data.provider || "Gemini AI");

      // Auto-detected info from API (if applicable)
      if (sourceLangCode === "auto") {
        setDetectedLang(data.detectedLanguage || "Detected Language");
        setDetectedCode(data.detectedCode || null);
      } else {
        setDetectedLang(null);
        setDetectedCode(null);
      }

      // Add to translation history
      const newItem: HistoryItem = {
        id: Date.now().toString(),
        sourceText: textToTranslate,
        translatedText: data.translatedText,
        sourceLang: currentSourceLanguage.name,
        targetLang: currentTargetLanguage.name,
        sourceCode: sourceLangCode,
        targetCode: targetLangCode,
        timestamp: Date.now(),
      };

      // Filter out duplicates of exact same translation to keep history neat
      const filteredHistory = history.filter(
        (item) => !(item.sourceText === textToTranslate && item.targetCode === targetLangCode)
      );
      saveHistoryToStorage([newItem, ...filteredHistory].slice(0, 50)); // Limit to 50 items

    } catch (err: any) {
      console.error("Translation request failed:", err);
      setError(err.message || "An unexpected error occurred. Please try again.");
    } finally {
      setIsLoading(false);
    }
  };

  // Trigger translation when pressing Enter with Cmd/Ctrl or clicking button
  const handleKeyDown = (e: React.KeyboardEvent<HTMLTextAreaElement>) => {
    if (e.key === "Enter" && (e.metaKey || e.ctrlKey)) {
      e.preventDefault();
      handleTranslate();
    }
  };

  // Swap source and target text/languages
  const handleSwap = () => {
    // If source is auto, we swap target to source, and default target to previous detected lang or English
    if (sourceLangCode === "auto") {
      const detectedOrEnglish = detectedCode || "en";
      const tempTargetText = targetText;
      const tempSourceText = sourceText;

      setSourceLangCode(targetLangCode);
      setTargetLangCode(detectedOrEnglish);
      setSourceText(tempTargetText);
      setTargetText(tempSourceText);
    } else {
      const tempSourceCode = sourceLangCode;
      const tempTargetCode = targetLangCode;
      const tempSourceText = sourceText;
      const tempTargetText = targetText;

      setSourceLangCode(tempTargetCode);
      setTargetLangCode(tempSourceCode);
      setSourceText(tempTargetText);
      setTargetText(tempSourceText);
    }
    // Reset provider/detected info on swap
    setTranslationProvider(null);
    setDetectedLang(null);
    setDetectedCode(null);
  };

  // Copy text to clipboard with animation feedback
  const handleCopy = async (text: string, isSource: boolean) => {
    if (!text) return;
    try {
      await navigator.clipboard.writeText(text);
      if (isSource) {
        setSourceCopied(true);
        setTimeout(() => setSourceCopied(false), 2000);
      } else {
        setTargetCopied(true);
        setTimeout(() => setTargetCopied(false), 2000);
      }
    } catch (err) {
      console.error("Failed to copy text:", err);
    }
  };

  // Text-To-Speech (TTS) integration
  const handleSpeak = (text: string, langCode: string, isSource: boolean) => {
    if (!text || !synthRef.current) return;

    // If currently speaking, stop it
    if (synthRef.current.speaking) {
      synthRef.current.cancel();
      setIsSpeakingSource(false);
      setIsSpeakingTarget(false);
      
      // If we clicked speaker again, we just toggle it off
      if (
        (isSource && isSpeakingSource) ||
        (!isSource && isSpeakingTarget)
      ) {
        return;
      }
    }

    const utterance = new SpeechSynthesisUtterance(text);
    
    // Attempt to match best voice based on speechCode
    const voices = synthRef.current.getVoices();
    const matchedVoice = voices.find(
      (v) => v.lang.toLowerCase() === langCode.toLowerCase() || v.lang.startsWith(langCode.split("-")[0])
    );
    if (matchedVoice) {
      utterance.voice = matchedVoice;
    }

    utterance.lang = langCode;

    utterance.onstart = () => {
      if (isSource) setIsSpeakingSource(true);
      else setIsSpeakingTarget(true);
    };

    utterance.onend = () => {
      if (isSource) setIsSpeakingSource(false);
      else setIsSpeakingTarget(false);
    };

    utterance.onerror = (e) => {
      console.error("Speech Synthesis Error:", e);
      if (isSource) setIsSpeakingSource(false);
      else setIsSpeakingTarget(false);
    };

    currentUtteranceRef.current = utterance;
    synthRef.current.speak(utterance);
  };

  // Stop any active speech
  const stopSpeech = () => {
    if (synthRef.current) {
      synthRef.current.cancel();
      setIsSpeakingSource(false);
      setIsSpeakingTarget(false);
    }
  };

  // Restore item from translation history
  const restoreHistoryItem = (item: HistoryItem) => {
    setSourceText(item.sourceText);
    setTargetText(item.translatedText);
    setSourceLangCode(item.sourceCode);
    setTargetLangCode(item.targetCode);
    setError(null);
    setDetectedLang(null);
    setTranslationProvider(null);
  };

  // Delete single item from history
  const deleteHistoryItem = (e: React.MouseEvent, id: string) => {
    e.stopPropagation();
    const updated = history.filter((item) => item.id !== id);
    saveHistoryToStorage(updated);
  };

  // Clear all history
  const clearAllHistory = () => {
    if (window.confirm("Are you sure you want to clear your translation history?")) {
      saveHistoryToStorage([]);
    }
  };

  // Format time for history entries
  const formatTime = (timestamp: number) => {
    const date = new Date(timestamp);
    return date.toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" });
  };

  return (
    <div className="min-h-screen bg-[#020617] text-[#f8fafc] font-sans flex flex-col justify-between selection:bg-blue-600/50 selection:text-white relative">
      
      {/* Subtle ambient lighting glows to enrich the Professional Polish feel */}
      <div className="absolute top-0 left-1/3 w-[500px] h-[500px] bg-blue-600/5 rounded-full filter blur-[120px] pointer-events-none" />
      <div className="absolute bottom-20 right-1/4 w-[400px] h-[400px] bg-indigo-500/5 rounded-full filter blur-[100px] pointer-events-none" />

      {/* Main Container */}
      <div className="container mx-auto px-4 py-8 max-w-5xl z-10 flex-grow flex flex-col justify-center">
        
        {/* Header - Styled to match Linguist AI branding */}
        <header className="text-center mb-8">
          <motion.div 
            initial={{ opacity: 0, y: -10 }}
            animate={{ opacity: 1, y: 0 }}
            className="inline-flex items-center gap-3 px-4 py-2 rounded-full bg-[#0f172a]/80 border border-[#1e293b] backdrop-blur-md mb-4 shadow-xl"
          >
            <div className="w-5 h-5 bg-blue-600 rounded-md flex items-center justify-center text-white font-bold text-xs shadow-md">
              <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="3" strokeLinecap="round" strokeLinejoin="round">
                <path d="M5 8l6 6"></path>
                <path d="M4 14l6-6 2-3"></path>
                <path d="M2 5h12"></path>
                <path d="M7 2h1"></path>
                <path d="M22 22l-5-10-5 10"></path>
                <path d="M14 18h6"></path>
              </svg>
            </div>
            <span className="text-xs font-semibold tracking-wider uppercase text-blue-400 font-mono">
              Linguist AI
            </span>
            <span className="text-slate-600">•</span>
            <div className="flex items-center gap-1.5">
              <div className="w-1.5 h-1.5 bg-emerald-500 rounded-full animate-pulse"></div>
              <span className="text-[10px] font-mono text-slate-400">Neural Engine Connected</span>
            </div>
          </motion.div>
          
          <motion.h1 
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            transition={{ delay: 0.1 }}
            className="text-4xl md:text-5xl font-display font-bold tracking-tight bg-gradient-to-r from-slate-50 via-slate-100 to-slate-400 bg-clip-text text-transparent"
          >
            Language Translation Tool
          </motion.h1>
          <motion.p 
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            transition={{ delay: 0.2 }}
            className="text-[#94a3b8] mt-2 text-sm max-w-md mx-auto"
          >
            Professional, zero-latency machine translation powered by Google Gemini.
          </motion.p>
        </header>

        {/* Translation Widget Card */}
        <main className="w-full">
          <div className="bg-[#0f172a] border border-[#1e293b] rounded-3xl shadow-2xl p-5 md:p-8 mb-6 transition-all duration-300">
            
            {/* Toolbar: Language Selectors and Swap button */}
            <div className="flex flex-col sm:flex-row items-center gap-4 mb-6 bg-[#1e293b]/40 p-3.5 rounded-xl border border-[#1e293b]/60">
              
              {/* Source Language Select */}
              <div className="w-full sm:flex-1 relative">
                <label className="absolute -top-2 left-3 px-1.5 bg-[#0f172a] text-[10px] font-mono text-[#94a3b8] uppercase tracking-wider rounded">
                  Source Language
                </label>
                <select
                  value={sourceLangCode}
                  onChange={(e) => {
                    setSourceLangCode(e.target.value);
                    setDetectedLang(null);
                    setDetectedCode(null);
                  }}
                  className="w-full h-11 pl-4 pr-10 rounded-xl bg-[#1e293b] text-[#f8fafc] border border-[#1e293b] focus:border-blue-500 focus:ring-1 focus:ring-blue-500 text-sm font-medium appearance-none outline-none cursor-pointer transition-all"
                >
                  <option value="auto">Auto-Detect Language</option>
                  {SUPPORTED_LANGUAGES.map((lang) => (
                    <option key={lang.code} value={lang.code}>
                      {lang.name}
                    </option>
                  ))}
                </select>
                <div className="absolute right-3 top-3.5 pointer-events-none text-slate-400 border-l border-slate-700/60 pl-2">
                  <svg className="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M19 9l-7 7-7-7" />
                  </svg>
                </div>
              </div>

              {/* Swap Button - Styled exactly like the design HTML swap-btn */}
              <motion.button
                whileHover={{ scale: 1.1, rotate: 180 }}
                whileTap={{ scale: 0.95 }}
                transition={{ type: "spring", stiffness: 300, damping: 20 }}
                onClick={handleSwap}
                className="w-10 h-10 rounded-full border border-[#1e293b] bg-[#0f172a] text-[#94a3b8] hover:border-blue-500 hover:text-blue-400 flex items-center justify-center cursor-pointer shadow-md transition-all shrink-0"
                title="Swap Languages"
              >
                <ArrowLeftRight className="w-4 h-4" />
              </motion.button>

              {/* Target Language Select */}
              <div className="w-full sm:flex-1 relative">
                <label className="absolute -top-2 left-3 px-1.5 bg-[#0f172a] text-[10px] font-mono text-[#94a3b8] uppercase tracking-wider rounded">
                  Target Language
                </label>
                <select
                  value={targetLangCode}
                  onChange={(e) => setTargetLangCode(e.target.value)}
                  className="w-full h-11 pl-4 pr-10 rounded-xl bg-[#1e293b] text-[#f8fafc] border border-[#1e293b] focus:border-blue-500 focus:ring-1 focus:ring-blue-500 text-sm font-medium appearance-none outline-none cursor-pointer transition-all"
                >
                  {SUPPORTED_LANGUAGES.map((lang) => (
                    <option key={lang.code} value={lang.code}>
                      {lang.name}
                    </option>
                  ))}
                </select>
                <div className="absolute right-3 top-3.5 pointer-events-none text-slate-400 border-l border-slate-700/60 pl-2">
                  <svg className="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M19 9l-7 7-7-7" />
                  </svg>
                </div>
              </div>

            </div>

            {/* Error Message */}
            <AnimatePresence>
              {error && (
                <motion.div
                  initial={{ opacity: 0, height: 0 }}
                  animate={{ opacity: 1, height: "auto" }}
                  exit={{ opacity: 0, height: 0 }}
                  className="mb-4 overflow-hidden"
                >
                  <div className="p-3 bg-red-500/10 border border-red-500/20 text-red-300 rounded-lg flex items-start gap-2.5 text-xs">
                    <AlertCircle className="w-4 h-4 shrink-0 text-red-400 mt-0.5" />
                    <div className="flex-grow">
                      <p className="font-semibold">Translation Error</p>
                      <p className="text-red-300/80 mt-0.5">{error}</p>
                    </div>
                    <button 
                      onClick={() => setError(null)} 
                      className="text-red-400 hover:text-red-200"
                    >
                      <X className="w-3.5 h-3.5" />
                    </button>
                  </div>
                </motion.div>
              )}
            </AnimatePresence>

            {/* Main Grid: Textareas */}
            <div className="grid grid-cols-1 md:grid-cols-2 gap-5">
              
              {/* Source Column */}
              <div className="flex flex-col bg-[#1e293b] rounded-2xl border border-[#1e293b] p-4.5 transition-all">
                <div className="flex items-center justify-between mb-3">
                  <span className="text-xs font-semibold text-[#94a3b8] tracking-wide flex items-center gap-1.5">
                    {currentSourceLanguage.name}
                    {detectedLang && (
                      <span className="px-1.5 py-0.5 rounded bg-blue-500/10 text-blue-400 font-mono text-[10px] font-medium border border-blue-500/20 animate-pulse">
                        {detectedLang}
                      </span>
                    )}
                  </span>
                  
                  {sourceText && (
                    <button
                      onClick={() => {
                        setSourceText("");
                        setTargetText("");
                        setDetectedLang(null);
                        setDetectedCode(null);
                        stopSpeech();
                      }}
                      className="p-1 rounded-md text-[#94a3b8] hover:text-[#f8fafc] hover:bg-[#0f172a]/60 transition-colors cursor-pointer"
                      title="Clear Text"
                    >
                      <X className="w-3.5 h-3.5" />
                    </button>
                  )}
                </div>

                <textarea
                  id="source-textarea"
                  value={sourceText}
                  onChange={(e) => setSourceText(e.target.value.slice(0, 5000))}
                  onKeyDown={handleKeyDown}
                  placeholder="Enter text to translate..."
                  className="w-full h-44 md:h-56 bg-transparent text-[#f8fafc] placeholder-slate-500 resize-none outline-none border-none text-lg leading-relaxed font-sans"
                  maxLength={5000}
                />

                {/* Character Counter & Input Actions */}
                <div className="flex items-center justify-between pt-3 mt-2 border-t border-slate-700/20">
                  <div className="flex gap-2">
                    {/* Speech Synthesis Button */}
                    <button
                      onClick={() => handleSpeak(sourceText, currentSourceLanguage.speechCode, true)}
                      disabled={!sourceText}
                      className={`p-2 rounded-lg transition-all ${
                        sourceText 
                          ? isSpeakingSource 
                            ? "bg-rose-500/20 text-rose-400 hover:bg-rose-500/30 cursor-pointer"
                            : "bg-[#0f172a]/40 hover:bg-[#0f172a]/80 text-[#94a3b8] hover:text-[#f8fafc] cursor-pointer" 
                          : "opacity-40 cursor-not-allowed text-[#94a3b8]"
                      }`}
                      title={isSpeakingSource ? "Stop Speech" : "Text-to-speech"}
                    >
                      {isSpeakingSource ? <VolumeX className="w-4.5 h-4.5" /> : <Volume2 className="w-4.5 h-4.5" />}
                    </button>

                    {/* Copy Button */}
                    <button
                      onClick={() => handleCopy(sourceText, true)}
                      disabled={!sourceText}
                      className={`p-2 rounded-lg transition-all ${
                        sourceText 
                          ? "bg-[#0f172a]/40 hover:bg-[#0f172a]/80 text-[#94a3b8] hover:text-[#f8fafc] cursor-pointer" 
                          : "opacity-40 cursor-not-allowed text-[#94a3b8]"
                      }`}
                      title="Copy to clipboard"
                    >
                      {sourceCopied ? <Check className="w-4.5 h-4.5 text-emerald-400" /> : <Copy className="w-4.5 h-4.5" />}
                    </button>
                  </div>

                  <span className="text-[11px] font-mono text-[#94a3b8] tracking-wider">
                    {sourceText.length.toLocaleString()} / 5,000
                  </span>
                </div>
              </div>

              {/* Target Column - Styled with slightly darker read-only background */}
              <div className="flex flex-col bg-[#1e293b]/50 rounded-2xl border border-[#1e293b] p-4.5 relative transition-all">
                <div className="flex items-center justify-between mb-3">
                  <span className="text-xs font-semibold text-[#94a3b8] tracking-wide">
                    {currentTargetLanguage.name}
                  </span>
                  
                  {translationProvider && !isLoading && (
                    <span className="text-[10px] font-mono text-[#94a3b8] flex items-center gap-1.5 bg-[#0f172a] px-2 py-0.5 rounded-full border border-slate-800">
                      <Sparkles className="w-3 h-3 text-blue-400" /> {translationProvider}
                    </span>
                  )}
                </div>

                <div className="relative flex-grow flex flex-col justify-between h-44 md:h-56">
                  {isLoading ? (
                    <div className="absolute inset-0 flex flex-col items-center justify-center bg-[#0f172a]/40 backdrop-blur-[1px] z-20 rounded-lg">
                      <div className="w-8 h-8 rounded-full border-2 border-blue-500/20 border-t-blue-500 animate-spin mb-3" />
                      <p className="text-xs font-mono text-blue-400 animate-pulse">Translating source text...</p>
                    </div>
                  ) : null}

                  <textarea
                    readOnly
                    value={targetText}
                    placeholder="Translation will appear here..."
                    className="w-full h-full bg-transparent text-[#f8fafc] placeholder-slate-500 resize-none outline-none border-none text-lg leading-relaxed font-sans"
                  />
                </div>

                {/* Output Actions */}
                <div className="flex items-center justify-between pt-3 mt-2 border-t border-slate-700/20">
                  <div className="flex gap-2">
                    {/* Speech Synthesis Button */}
                    <button
                      onClick={() => handleSpeak(targetText, currentTargetLanguage.speechCode, false)}
                      disabled={!targetText}
                      className={`p-2 rounded-lg transition-all ${
                        targetText 
                          ? isSpeakingTarget 
                            ? "bg-rose-500/20 text-rose-400 hover:bg-rose-500/30 cursor-pointer"
                            : "bg-[#0f172a]/40 hover:bg-[#0f172a]/80 text-[#94a3b8] hover:text-[#f8fafc] cursor-pointer" 
                          : "opacity-40 cursor-not-allowed text-[#94a3b8]"
                      }`}
                      title={isSpeakingTarget ? "Stop Speech" : "Text-to-speech"}
                    >
                      {isSpeakingTarget ? <VolumeX className="w-4.5 h-4.5" /> : <Volume2 className="w-4.5 h-4.5" />}
                    </button>

                    {/* Copy Button */}
                    <button
                      onClick={() => handleCopy(targetText, false)}
                      disabled={!targetText}
                      className={`p-2 rounded-lg transition-all ${
                        targetText 
                          ? "bg-[#0f172a]/40 hover:bg-[#0f172a]/80 text-[#94a3b8] hover:text-[#f8fafc] cursor-pointer" 
                          : "opacity-40 cursor-not-allowed text-[#94a3b8]"
                      }`}
                      title="Copy to clipboard"
                    >
                      {targetCopied ? <Check className="w-4.5 h-4.5 text-emerald-400" /> : <Copy className="w-4.5 h-4.5" />}
                    </button>
                  </div>

                  {targetText && (
                    <span className="text-[10px] font-mono text-emerald-400/80 flex items-center gap-1 bg-emerald-500/5 px-2 py-0.5 rounded-full border border-emerald-500/10">
                      <Check className="w-3.5 h-3.5" /> Translated
                    </span>
                  )}
                </div>
              </div>

            </div>

            {/* Translate Button & Controls footer */}
            <div className="mt-8 flex flex-col sm:flex-row items-center justify-between gap-4 pt-6 border-t border-[#1e293b]">
              <div className="text-xs text-[#94a3b8] opacity-80 font-mono flex items-center gap-2">
                <span className="w-1.5 h-1.5 bg-blue-500 rounded-full"></span>
                <span>Press Ctrl + Enter to translate</span>
              </div>

              <div className="flex flex-col sm:flex-row items-center gap-3 w-full sm:w-auto">
                {/* History Toggle Button */}
                <button
                  onClick={() => setShowHistory(!showHistory)}
                  className={`h-12 px-4 rounded-xl border flex items-center justify-center gap-2 cursor-pointer transition-all shrink-0 w-full sm:w-auto text-sm font-medium ${
                    showHistory 
                      ? "bg-[#1e293b] text-blue-400 border-blue-500/30" 
                      : "bg-[#1e293b]/40 text-[#94a3b8] border-[#1e293b] hover:bg-[#1e293b]/80"
                  }`}
                  title="Translation History"
                >
                  <History className="w-4.5 h-4.5" />
                  <span>History ({history.length})</span>
                </button>

                {/* Primary Translate Button */}
                <button
                  onClick={() => handleTranslate()}
                  disabled={isLoading || !sourceText.trim()}
                  className={`h-12 px-8 rounded-xl text-sm font-semibold tracking-wide flex items-center justify-center gap-2 shadow-lg transition-all relative overflow-hidden w-full sm:w-auto ${
                    isLoading || !sourceText.trim()
                      ? "bg-slate-800 text-slate-500 border border-slate-700/50 cursor-not-allowed shadow-none"
                      : "bg-blue-600 hover:bg-blue-500 text-white active:scale-[0.98] cursor-pointer"
                  }`}
                >
                  {isLoading ? (
                    <>
                      <div className="w-4 h-4 rounded-full border-2 border-white/20 border-t-white animate-spin" />
                      <span>Translating...</span>
                    </>
                  ) : (
                    <span>Translate Text</span>
                  )}
                </button>
              </div>
            </div>

          </div>
        </main>

        {/* Expandable History Panel */}
        <AnimatePresence>
          {showHistory && (
            <motion.section
              initial={{ opacity: 0, y: 15 }}
              animate={{ opacity: 1, y: 0 }}
              exit={{ opacity: 0, y: 15 }}
              className="bg-[#0f172a]/80 border border-[#1e293b] rounded-3xl backdrop-blur-md p-5 md:p-6 mb-6"
            >
              <div className="flex items-center justify-between mb-4 pb-3 border-b border-[#1e293b]">
                <div className="flex items-center gap-2">
                  <Clock className="w-4 h-4 text-blue-400" />
                  <h3 className="font-display font-semibold text-sm text-[#f8fafc]">Recent Translations</h3>
                </div>
                {history.length > 0 ? (
                  <button
                    onClick={clearAllHistory}
                    className="text-xs text-rose-400 hover:text-rose-300 flex items-center gap-1.5 px-2.5 py-1.5 rounded-lg hover:bg-rose-500/10 cursor-pointer transition-all"
                  >
                    <Trash2 className="w-3.5 h-3.5" />
                    <span>Clear All</span>
                  </button>
                ) : null}
              </div>

              {history.length === 0 ? (
                <div className="py-8 text-center">
                  <Clock className="w-8 h-8 text-slate-700 mx-auto mb-2.5" />
                  <p className="text-sm text-[#94a3b8]">Your translation history is empty.</p>
                  <p className="text-xs text-slate-600 mt-1">Translate some text above to build your history log.</p>
                </div>
              ) : (
                <div className="space-y-3 max-h-[300px] overflow-y-auto pr-1">
                  {history.map((item) => (
                    <div
                      key={item.id}
                      onClick={() => restoreHistoryItem(item)}
                      className="group p-3 rounded-xl bg-[#1e293b]/40 hover:bg-blue-600/[0.04] hover:border-blue-500/20 border border-slate-900 cursor-pointer flex flex-col justify-between md:flex-row md:items-center gap-3 transition-all"
                    >
                      <div className="flex-grow space-y-1">
                        <div className="flex items-center gap-2 flex-wrap text-[10px] font-mono text-slate-500">
                          <span className="text-[#94a3b8] font-semibold">{item.sourceLang}</span>
                          <span>→</span>
                          <span className="text-blue-400 font-semibold">{item.targetLang}</span>
                          <span className="text-slate-600">•</span>
                          <span>{formatTime(item.timestamp)}</span>
                        </div>
                        <p className="text-xs text-slate-300 font-normal line-clamp-1 group-hover:text-slate-100">
                          {item.sourceText}
                        </p>
                        <p className="text-xs text-[#94a3b8] font-normal line-clamp-1 group-hover:text-slate-200">
                          {item.translatedText}
                        </p>
                      </div>
                      
                      <div className="flex items-center gap-2 shrink-0 self-end md:self-auto">
                        <button
                          onClick={(e) => {
                            e.stopPropagation();
                            navigator.clipboard.writeText(item.translatedText);
                          }}
                          className="p-1.5 rounded bg-[#1e293b] border border-[#1e293b] hover:text-blue-400 text-[#94a3b8] transition-colors cursor-pointer"
                          title="Copy Translation"
                        >
                          <Copy className="w-3.5 h-3.5" />
                        </button>
                        <button
                          onClick={(e) => deleteHistoryItem(e, item.id)}
                          className="p-1.5 rounded bg-[#1e293b] border border-[#1e293b] hover:text-rose-400 text-[#94a3b8] transition-colors cursor-pointer"
                          title="Delete"
                        >
                          <Trash2 className="w-3.5 h-3.5" />
                        </button>
                      </div>
                    </div>
                  ))}
                </div>
              )}
            </motion.section>
          )}
        </AnimatePresence>

      </div>

      {/* Footer */}
      <footer className="py-6 border-t border-[#1e293b] bg-[#020617] backdrop-blur-sm z-10 text-center">
        <p className="text-[#94a3b8]/50 text-[11px] font-mono tracking-wide">
          Designed with pure typography & negative space • Local cache storage enabled
        </p>
      </footer>

    </div>
  );
}
