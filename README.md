import React, { useState, useEffect } from 'react';
import { 
  Upload, FileImage, CheckCircle, AlertTriangle, 
  Layout, Palette, Brain, ChevronRight, BarChart3, 
  Loader2, RefreshCw, Sparkles, Download, X,
  Lock, Crown, Zap, MousePointerClick
} from 'lucide-react';

// --- System Configuration ---
const apiKey = ""; // API Key injected by environment

export default function App() {
  const [step, setStep] = useState<'upload' | 'analyzing' | 'result'>('upload');
  const [file, setFile] = useState<File | null>(null);
  const [imagePreview, setImagePreview] = useState<string | null>(null);
  const [analysisResult, setAnalysisResult] = useState<any>(null);
  const [error, setError] = useState<string | null>(null);
  
  // --- Monetization State ---
  const [isPro, setIsPro] = useState(false); // Toggle to simulate Pro user
  const [showPaywall, setShowPaywall] = useState(false);
  const [isGeneratingPDF, setIsGeneratingPDF] = useState(false);

  // --- Gemini API Handler ---
  const analyzeImageWithGemini = async (base64Image: string) => {
    try {
      // 1. Prepare the payload
      const prompt = `
        You are a strict and professional Architecture & Interior Design Professor. 
        Analyze the attached image critically. It could be an architecture portfolio page or a presentation panel (board).
        
        Return the response in strictly valid JSON format without any markdown code blocks.
        The JSON structure must be:
        {
          "score": <integer 0-100>,
          "metrics": {
            "structure": <integer 0-100, logic flow and hierarchy>,
            "storytelling": <integer 0-100>,
            "visual": <integer 0-100, aesthetics and composition>,
            "technical": <integer 0-100, drawing quality and readability>,
            "creativity": <integer 0-100>
          },
          "critique": [
            { "type": "warning", "title": "<short title>", "desc": "<detailed critique in Korean>" },
            { "type": "good", "title": "<short title>", "desc": "<detailed praise in Korean>" },
            { "type": "suggestion", "title": "<short title>", "desc": "<specific improvement advice in Korean>" }
          ],
          "layoutFeedback": "<Detailed analysis of current layout, grid system, and information hierarchy in Korean>",
          "colorPalette": [
            { "hex": "<hex code>", "name": "<color name>" },
            { "hex": "<hex code>", "name": "<color name>" },
            { "hex": "<hex code>", "name": "<color name>" },
            { "hex": "<hex code>", "name": "<color name>" }
          ],
          "proTips": [
             "<Advanced architectural advice 1>",
             "<Advanced architectural advice 2>"
          ]
        }
      `;

      const matches = base64Image.match(/^data:(.+);base64,(.+)$/);
      if (!matches || matches.length !== 3) {
        throw new Error("Invalid image format");
      }
      const mimeType = matches[1];
      const base64Data = matches[2];

      const response = await fetch(
        `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`,
        {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            contents: [{
              role: "user",
              parts: [
                { text: prompt },
                { inlineData: { mimeType: mimeType, data: base64Data } }
              ]
            }]
          })
        }
      );

      if (!response.ok) throw new Error(`API Request Failed`);

      const data = await response.json();
      const textResponse = data.candidates?.[0]?.content?.parts?.[0]?.text;
      
      if (!textResponse) throw new Error("No response content from AI");

      const jsonString = textResponse.replace(/```json/g, '').replace(/```/g, '').trim();
      return JSON.parse(jsonString);

    } catch (err: any) {
      console.error(err);
      throw new Error(err.message || "AI 분석에 실패했습니다.");
    }
  };

  const handleFileUpload = (e: React.ChangeEvent<HTMLInputElement>) => {
    if (e.target.files && e.target.files[0]) {
      processFile(e.target.files[0]);
    }
  };

  const processFile = (file: File) => {
    const reader = new FileReader();
    reader.onloadend = () => {
      setFile(file);
      setImagePreview(reader.result as string);
      setStep('analyzing');
      setError(null);
      analyzeImageWithGemini(reader.result as string)
        .then(result => {
          setAnalysisResult(result);
          setStep('result');
        })
        .catch(err => {
          setError(err.message);
          setStep('upload');
        });
    };
    reader.readAsDataURL(file);
  };

  const handleDownloadPDF = () => {
    if (!isPro) {
      setShowPaywall(true);
      return;
    }

    setIsGeneratingPDF(true);

    // Dynamically load html2pdf.js
    const script = document.createElement('script');
    script.src = "https://cdnjs.cloudflare.com/ajax/libs/html2pdf.js/0.10.1/html2pdf.bundle.min.js";
    script.onload = () => {
      const element = document.getElementById('report-container');
      const opt = {
        margin:       [10, 10, 10, 10],
        filename:     `ArchCritique_${file?.name || 'Report'}.pdf`,
        image:        { type: 'jpeg', quality: 0.98 },
        html2canvas:  { scale: 2, useCORS: true, logging: false },
        jsPDF:        { unit: 'mm', format: 'a4', orientation: 'portrait' }
      };
      
      // @ts-ignore
      window.html2pdf().set(opt).from(element).save().then(() => {
        setIsGeneratingPDF(false);
      });
    };
    script.onerror = () => {
      alert("PDF 생성 라이브러리 로드에 실패했습니다.");
      setIsGeneratingPDF(false);
    }
    document.body.appendChild(script);
  };

  const handleDragOver = (e: React.DragEvent) => { e.preventDefault(); };
  const handleDrop = (e: React.DragEvent) => {
    e.preventDefault();
    if (e.dataTransfer.files && e.dataTransfer.files[0]) {
      processFile(e.dataTransfer.files[0]);
    }
  };

  // --- Components ---

  const Header = () => (
    <header className="flex items-center justify-between px-6 py-4 bg-slate-900 text-white border-b border-slate-800 sticky top-0 z-50 no-print">
      <div className="flex items-center space-x-2 cursor-pointer" onClick={() => setStep('upload')}>
        <Brain className="w-6 h-6 text-emerald-400" />
        <span className="text-xl font-bold tracking-tight">ArchCritique.AI <span className="text-xs bg-emerald-600 px-1 rounded ml-1">BETA</span></span>
      </div>
      
      <div className="flex items-center gap-4">
        {/* Simulator Toggle */}
        <div className="flex items-center bg-slate-800 rounded-full p-1 px-3 border border-slate-700">
          <span className="text-xs text-slate-400 mr-2">Simulate:</span>
          <button 
            onClick={() => setIsPro(!isPro)}
            className={`text-xs font-bold px-3 py-1 rounded-full transition-all ${isPro ? 'bg-amber-400 text-slate-900' : 'bg-slate-600 text-slate-300'}`}
          >
            {isPro ? 'PRO USER' : 'FREE USER'}
          </button>
        </div>

        {!isPro && (
          <button 
            onClick={() => setShowPaywall(true)}
            className="flex items-center px-4 py-2 bg-gradient-to-r from-amber-500 to-yellow-500 text-slate-900 text-sm font-bold rounded-lg hover:shadow-lg hover:shadow-amber-500/20 transition-all"
          >
            <Crown className="w-4 h-4 mr-1" /> Upgrade Pro
          </button>
        )}
      </div>
    </header>
  );

  const PaywallModal = () => {
    if (!showPaywall) return null;
    return (
      <div className="fixed inset-0 bg-slate-900/80 backdrop-blur-sm z-50 flex items-center justify-center p-4" style={{ zIndex: 9999 }}>
        <div className="bg-white rounded-2xl shadow-2xl max-w-md w-full overflow-hidden animate-in fade-in zoom-in duration-200">
          <div className="bg-gradient-to-br from-slate-900 to-slate-800 p-6 text-white text-center relative overflow-hidden">
            <div className="absolute top-0 right-0 p-4 opacity-10"><Crown className="w-32 h-32" /></div>
            <h2 className="text-2xl font-bold mb-2 flex justify-center items-center gap-2">
              <Crown className="w-6 h-6 text-amber-400" /> Unlock ArchCritique Pro
            </h2>
            <p className="text-slate-300 text-sm">교수님급 크리틱을 무제한으로 받아보세요.</p>
          </div>
          <div className="p-6 space-y-4">
            {[
              "AI 레퍼런스 이미지 검색 & 매칭",
              "레이아웃 자동 수정 제안 (Auto-Fix)",
              "심층 기술 분석 (Deep Technical Scan)",
              "고화질 PDF 리포트 다운로드"
            ].map((feature, idx) => (
              <div key={idx} className="flex items-center text-slate-700">
                <CheckCircle className="w-5 h-5 text-emerald-500 mr-3 flex-shrink-0" />
                <span className="font-medium">{feature}</span>
              </div>
            ))}
            <button 
              onClick={() => { setIsPro(true); setShowPaywall(false); }}
              className="w-full py-3 bg-slate-900 text-white font-bold rounded-xl mt-4 hover:bg-slate-800 transition flex justify-center items-center gap-2"
            >
              <Zap className="w-4 h-4 fill-amber-400 text-amber-400" /> 월 9,900원으로 시작하기
            </button>
            <button 
              onClick={() => setShowPaywall(false)}
              className="w-full py-2 text-slate-500 text-sm hover:text-slate-800 transition"
            >
              괜찮습니다, 제한된 기능으로 계속할게요.
            </button>
          </div>
        </div>
      </div>
    );
  };

  const UploadSection = () => (
    <div className="flex flex-col items-center justify-center min-h-[80vh] bg-slate-50 p-6">
      {/* 텍스트 줄바꿈 방지를 위해 max-w-2xl -> max-w-3xl로 변경 */}
      <div className="text-center mb-10 max-w-3xl">
        {/* 폰트 적용을 위해 font-gmarket 클래스 추가, 단어 단위 줄바꿈을 위해 break-keep 추가 */}
        <h1 className="text-4xl font-extrabold text-slate-900 mb-4 font-gmarket leading-snug break-keep">
          건축 디자인/패널을 <span className="text-emerald-600">지금 바로</span> 크리틱합니다
        </h1>
        <p className="text-lg text-slate-600">
          작업 이미지를 업로드하세요. AI가<br />
          포트폴리오와 패널의 레이아웃, 논리, 디자인을 분석합니다.
        </p>
      </div>

      <div 
        className="w-full max-w-xl h-64 border-2 border-dashed border-slate-300 rounded-2xl bg-white flex flex-col items-center justify-center cursor-pointer hover:border-emerald-500 hover:bg-emerald-50/30 transition-all duration-300 shadow-sm group relative"
        onDragOver={handleDragOver}
        onDrop={handleDrop}
      >
        <label className="w-full h-full flex flex-col items-center justify-center cursor-pointer z-10">
          <div className="p-4 bg-emerald-100 rounded-full text-emerald-600 mb-4 group-hover:scale-110 transition-transform">
            <Upload className="w-8 h-8" />
          </div>
          <p className="text-lg font-semibold text-slate-700">이미지 업로드 (포트폴리오/패널)</p>
          <p className="text-sm text-slate-500 mt-2">고해상도 JPG, PNG 권장</p>
          <input type="file" className="hidden" accept="image/*" onChange={handleFileUpload} />
        </label>
        {error && (
          <div className="absolute bottom-4 text-red-500 text-sm flex items-center bg-red-50 px-3 py-1 rounded-full border border-red-200">
            <AlertTriangle className="w-4 h-4 mr-1" /> {error}
          </div>
        )}
      </div>
      
      {!isPro && (
        <div className="mt-8 flex items-center gap-2 text-slate-400 text-sm bg-white px-4 py-2 rounded-full shadow-sm border border-slate-200">
          <Lock className="w-3 h-3" />
          <span>무료 버전은 일 3회 분석으로 제한됩니다.</span>
        </div>
      )}
    </div>
  );

  const AnalyzingScreen = () => (
    <div className="flex flex-col items-center justify-center min-h-[80vh] bg-slate-900 text-white p-6">
      <div className="w-full max-w-md space-y-8 text-center">
        <div className="relative inline-block">
          <div className="absolute inset-0 bg-emerald-500 blur-xl opacity-20 animate-pulse"></div>
          <img src={imagePreview!} alt="Analyzing" className="w-48 h-auto rounded-lg border border-slate-700 shadow-2xl mx-auto relative z-10 opacity-80" />
          <div className="absolute inset-0 flex items-center justify-center z-20">
             <div className="w-full h-1 bg-emerald-500/50 absolute top-1/2 animate-scan"></div>
          </div>
        </div>
        
        <div className="space-y-2">
          <h2 className="text-2xl font-bold flex items-center justify-center gap-2">
            <Loader2 className="w-6 h-6 animate-spin text-emerald-400" />
            AI가 이미지를 해석 중입니다...
          </h2>
          <p className="text-slate-400 text-sm">
            {isPro ? "Pro 엔진으로 정밀 분석 중입니다..." : "기본 분석 엔진 가동 중..."}
          </p>
        </div>
      </div>
    </div>
  );

  const ResultDashboard = () => {
    if (!analysisResult) return null;

    return (
      <div className="min-h-screen bg-slate-50 pb-20">
        <div className="max-w-7xl mx-auto px-4 py-8">
          
          {/* Header Area - Hidden in PDF */}
          <div className="flex flex-col md:flex-row justify-between items-start md:items-center mb-8 gap-4" data-html2canvas-ignore="true">
            <div>
              <div className="flex items-center gap-2 mb-1">
                <span className={`text-xs font-bold px-2 py-0.5 rounded flex items-center gap-1 ${isPro ? 'bg-amber-100 text-amber-800' : 'bg-emerald-100 text-emerald-800'}`}>
                  {isPro && <Crown className="w-3 h-3" />}
                  {isPro ? 'PREMIUM ANALYSIS' : 'BASIC ANALYSIS'}
                </span>
                <h2 className="text-2xl font-bold text-slate-900">진단 리포트</h2>
              </div>
              <p className="text-slate-500 text-sm">Target: {file?.name}</p>
            </div>
            <div className="flex gap-2">
              <button 
                onClick={() => setStep('upload')}
                className="flex items-center px-4 py-2 border border-slate-300 bg-white text-slate-700 rounded-lg hover:bg-slate-50 transition shadow-sm text-sm"
              >
                <RefreshCw className="w-4 h-4 mr-2" /> 재분석
              </button>
              <button 
                onClick={handleDownloadPDF}
                disabled={isGeneratingPDF}
                className={`flex items-center px-4 py-2 rounded-lg transition shadow-sm text-sm font-medium ${isPro ? 'bg-slate-900 text-white hover:bg-slate-800' : 'bg-slate-200 text-slate-400 cursor-not-allowed'}`}
              >
                {isGeneratingPDF ? <Loader2 className="w-4 h-4 mr-2 animate-spin"/> : (isPro ? <Download className="w-4 h-4 mr-2" /> : <Lock className="w-4 h-4 mr-2" />)}
                {isGeneratingPDF ? "생성 중..." : "PDF 저장"}
              </button>
            </div>
          </div>

          {/* Report Container for PDF Generation */}
          <div id="report-container" className="p-1">
            {/* PDF Header (Only visible in PDF, usually implied by layout but adding padding helps) */}
            
            <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
              
              {/* Left Column: Image & Score */}
              <div className="space-y-6">
                {/* Original Image */}
                <div className="bg-white p-4 rounded-xl shadow-sm border border-slate-100 break-inside-avoid">
                  <h3 className="text-sm font-semibold text-slate-500 mb-3 uppercase tracking-wide">Uploaded Page</h3>
                  <img src={imagePreview!} alt="Original" className="w-full rounded border border-slate-200" />
                </div>

                {/* Score */}
                <div className="bg-white p-6 rounded-xl shadow-sm border border-slate-100 relative overflow-hidden break-inside-avoid">
                  {!isPro && (
                    <div className="absolute top-2 right-2 cursor-pointer" onClick={() => setShowPaywall(true)} data-html2canvas-ignore="true">
                      <div className="bg-amber-100 hover:bg-amber-200 text-amber-800 text-xs px-2 py-1 rounded-full flex items-center transition">
                          <Lock className="w-3 h-3 mr-1" /> 상세 점수 잠김
                      </div>
                    </div>
                  )}
                  <h3 className="text-lg font-semibold text-slate-800 mb-4 flex items-center">
                    <BarChart3 className="w-5 h-5 mr-2 text-slate-500" /> 종합 평가
                  </h3>
                  <div className="flex items-center justify-center mb-6 relative">
                    <svg className="w-32 h-32 transform -rotate-90">
                      <circle cx="64" cy="64" r="60" stroke="#E2E8F0" strokeWidth="8" fill="transparent" />
                      <circle 
                        cx="64" cy="64" r="60" 
                        stroke={analysisResult.score > 80 ? "#10B981" : analysisResult.score > 60 ? "#F59E0B" : "#EF4444"} 
                        strokeWidth="8" 
                        fill="transparent" 
                        strokeDasharray="377" 
                        strokeDashoffset={377 - (377 * analysisResult.score) / 100} 
                      />
                    </svg>
                    <span className="absolute text-3xl font-bold text-slate-800">{analysisResult.score}</span>
                  </div>
                  
                  <div className="space-y-3">
                    {Object.entries(analysisResult.metrics).map(([key, value]: [string, any], idx) => (
                      <div key={key} className={`flex items-center justify-between text-sm ${!isPro && idx > 2 ? 'opacity-30 blur-[2px]' : ''}`}>
                        <span className="capitalize text-slate-600 w-24">{key}</span>
                        <div className="flex-1 h-2 bg-slate-100 rounded-full mx-3">
                          <div 
                            className={`h-full rounded-full ${Number(value) > 70 ? 'bg-emerald-500' : 'bg-slate-400'}`} 
                            style={{ width: `${value}%` }} 
                          />
                        </div>
                        <span className="font-bold text-slate-700">{!isPro && idx > 2 ? '??' : value}</span>
                      </div>
                    ))}
                    {!isPro && (
                      <div className="text-center mt-2" data-html2canvas-ignore="true">
                          <button onClick={() => setShowPaywall(true)} className="text-xs text-amber-600 font-medium hover:underline">
                              모든 세부 지표 확인하기 →
                          </button>
                      </div>
                    )}
                  </div>
                </div>

                {/* Palette */}
                <div className="bg-white p-6 rounded-xl shadow-sm border border-slate-100 break-inside-avoid">
                  <h3 className="text-lg font-semibold text-slate-800 mb-4 flex items-center">
                    <Palette className="w-5 h-5 mr-2 text-slate-500" /> 감지된 컬러 팔레트
                  </h3>
                  <div className="flex space-x-2 mb-2">
                    {analysisResult.colorPalette?.map((color: any, idx: number) => (
                      <div key={idx} className="flex-1 flex flex-col items-center group cursor-pointer" title={color.name}>
                        <div className="w-full h-10 rounded shadow-sm border border-slate-100" style={{ backgroundColor: color.hex }} />
                        <span className="text-[10px] text-slate-400 font-mono mt-1">{color.hex}</span>
                      </div>
                    ))}
                  </div>
                </div>
              </div>

              {/* Right Column: Critique & Layout Analysis */}
              <div className="lg:col-span-2 space-y-6">
                
                {/* Detailed Critique */}
                <div className="bg-white p-6 rounded-xl shadow-sm border border-slate-100 break-inside-avoid">
                  <h3 className="text-lg font-semibold text-slate-800 mb-4 flex items-center">
                    <Brain className="w-5 h-5 mr-2 text-slate-500" /> AI 상세 피드백
                  </h3>
                  <div className="space-y-4">
                    {analysisResult.critique.slice(0, isPro ? 10 : 2).map((item: any, idx: number) => (
                      <div key={idx} className={`p-4 rounded-lg border-l-4 ${
                        item.type === 'warning' ? 'bg-orange-50 border-orange-400' :
                        item.type === 'suggestion' ? 'bg-blue-50 border-blue-400' :
                        'bg-emerald-50 border-emerald-400'
                      }`}>
                        <div className="flex items-start">
                          {item.type === 'warning' && <AlertTriangle className="w-5 h-5 text-orange-500 mt-0.5 mr-3 flex-shrink-0" />}
                          {item.type === 'suggestion' && <Sparkles className="w-5 h-5 text-blue-500 mt-0.5 mr-3 flex-shrink-0" />}
                          {item.type === 'good' && <CheckCircle className="w-5 h-5 text-emerald-500 mt-0.5 mr-3 flex-shrink-0" />}
                          <div>
                            <h4 className={`text-sm font-bold mb-1 ${
                              item.type === 'warning' ? 'text-orange-800' :
                              item.type === 'suggestion' ? 'text-blue-800' :
                              'text-emerald-800'
                            }`}>{item.title}</h4>
                            <p className="text-sm text-slate-700 leading-relaxed">{item.desc}</p>
                          </div>
                        </div>
                      </div>
                    ))}
                    
                    {/* Paywall Overlay for Critique */}
                    {!isPro && (
                      <div className="relative p-4 rounded-lg border border-slate-200 bg-slate-50 overflow-hidden" data-html2canvas-ignore="true">
                          <div className="filter blur-sm select-none opacity-50">
                              <h4 className="font-bold mb-1">레이아웃의 비례감이 다소 아쉽습니다...</h4>
                              <p>전체적인 그리드 시스템을 3단 구성으로 변경하면 훨씬 더...</p>
                          </div>
                          <div className="absolute inset-0 flex flex-col items-center justify-center bg-white/60">
                              <Lock className="w-6 h-6 text-slate-400 mb-2" />
                              <p className="text-slate-600 font-medium text-sm">나머지 3개의 심층 피드백 잠김</p>
                              <button onClick={() => setShowPaywall(true)} className="mt-2 text-xs bg-slate-900 text-white px-3 py-1.5 rounded-full hover:bg-slate-800">
                                  Pro에서 전체 보기
                              </button>
                          </div>
                      </div>
                    )}
                  </div>
                </div>

                {/* Layout AI & Advanced Tools */}
                <div className="bg-slate-900 text-white p-6 rounded-xl shadow-lg relative overflow-hidden break-inside-avoid">
                  <div className="flex justify-between items-center mb-4 relative z-10">
                    <h3 className="text-lg font-semibold flex items-center text-white">
                      <Layout className="w-5 h-5 mr-2 text-emerald-400" /> 레이아웃 & 레퍼런스
                    </h3>
                    {isPro && <span className="text-xs bg-amber-500 text-slate-900 px-2 py-0.5 rounded font-bold">PRO UNLOCKED</span>}
                  </div>
                  
                  <div className="p-4 bg-slate-800 rounded-lg border border-slate-700 text-slate-300 text-sm leading-7 relative z-10">
                    {analysisResult.layoutFeedback}
                  </div>

                  {/* Pro Feature Area */}
                  <div className="mt-4 grid grid-cols-2 gap-3 relative z-10">
                    <div 
                        onClick={() => !isPro && setShowPaywall(true)}
                        className={`h-32 rounded flex flex-col items-center justify-center border border-dashed transition-all cursor-pointer ${
                            isPro 
                            ? 'bg-slate-800 border-slate-600 hover:border-emerald-500' 
                            : 'bg-slate-800/50 border-slate-700 hover:bg-slate-800'
                        }`}
                    >
                        {isPro ? (
                            <>
                              <Layout className="w-8 h-8 text-emerald-400 mb-2" />
                              <span className="text-xs text-emerald-400 font-bold">레이아웃 자동 수정 제안</span>
                              <span className="text-[10px] text-slate-500 mt-1">AI가 재배치한 그리드 보기</span>
                            </>
                        ) : (
                            <>
                              <Lock className="w-6 h-6 text-slate-500 mb-2" />
                              <span className="text-xs text-slate-400">레이아웃 제안 (잠김)</span>
                            </>
                        )}
                    </div>

                    <div 
                        onClick={() => !isPro && setShowPaywall(true)}
                        className={`h-32 rounded flex flex-col items-center justify-center border border-dashed transition-all cursor-pointer ${
                          isPro 
                          ? 'bg-slate-800 border-slate-600 hover:border-blue-500' 
                          : 'bg-slate-800/50 border-slate-700 hover:bg-slate-800'
                      }`}
                    >
                        {isPro ? (
                            <>
                              <FileImage className="w-8 h-8 text-blue-400 mb-2" />
                              <span className="text-xs text-blue-400 font-bold">유사 사례 레퍼런스</span>
                              <span className="text-[10px] text-slate-500 mt-1">성공적인 패널 예시 3건</span>
                            </>
                        ) : (
                            <>
                              <Lock className="w-6 h-6 text-slate-500 mb-2" />
                              <span className="text-xs text-slate-400">레퍼런스 매칭 (잠김)</span>
                            </>
                        )}
                    </div>
                  </div>

                  {/* Paywall Overlay for Layout Section (Partial) */}
                  {!isPro && (
                      <div className="absolute inset-x-0 bottom-0 h-2/3 bg-gradient-to-t from-slate-900 via-slate-900/90 to-transparent z-0 flex items-end justify-center pb-8" data-html2canvas-ignore="true">
                          <button 
                              onClick={() => setShowPaywall(true)}
                              className="flex items-center gap-2 px-5 py-2.5 bg-amber-500 hover:bg-amber-400 text-slate-900 font-bold rounded-full shadow-lg shadow-amber-900/50 transition-all transform hover:scale-105"
                          >
                              <Crown className="w-4 h-4" /> 고급 분석 기능 잠금해제
                          </button>
                      </div>
                  )}
                </div>

              </div>
            </div>
          </div>
        </div>
        
        {/* Paywall Modal */}
        <PaywallModal />
      </div>
    );
  };

  return (
    <div className="min-h-screen bg-slate-50 font-sans text-slate-900">
      <Header />
      {step === 'upload' && <UploadSection />}
      {step === 'analyzing' && <AnalyzingScreen />}
      {step === 'result' && <ResultDashboard />}
      
      <style>{`
        /* Gmarket Sans Medium 폰트 임포트 */
        @font-face {
            font-family: 'GmarketSansMedium';
            src: url('https://cdn.jsdelivr.net/gh/projectnoonnu/noonfonts_2001@1.1/GmarketSansMedium.woff') format('woff');
            font-weight: normal;
            font-style: normal;
        }
        
        .font-gmarket {
            font-family: 'GmarketSansMedium', sans-serif;
        }
        
        /* 한국어 단어 단위 줄바꿈 유틸리티 */
        .break-keep {
            word-break: keep-all;
        }

        @keyframes scan {
          0% { top: 0%; opacity: 0; }
          20% { opacity: 1; }
          80% { opacity: 1; }
          100% { top: 100%; opacity: 0; }
        }
        .animate-scan {
          animation: scan 2s cubic-bezier(0.4, 0, 0.2, 1) infinite;
        }

        /* PDF 생성 시 페이지 넘김 방지 */
        .break-inside-avoid {
          page-break-inside: avoid;
        }
      `}</style>
    </div>
  );
}
