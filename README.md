# archfolio.ai

import React, { useState, useEffect, useRef } from 'react';
import { 
  Upload, FileImage, CheckCircle, AlertTriangle, 
  Layout, Palette, Brain, ChevronRight, BarChart3, 
  Loader2, RefreshCw, Sparkles, Download, X
} from 'lucide-react';

// --- System Configuration ---
const apiKey = ""; // API Key injected by environment

export default function App() {
  const [step, setStep] = useState<'upload' | 'analyzing' | 'result'>('upload');
  const [file, setFile] = useState<File | null>(null);
  const [imagePreview, setImagePreview] = useState<string | null>(null);
  const [analysisResult, setAnalysisResult] = useState<any>(null);
  const [error, setError] = useState<string | null>(null);

  // --- Gemini API Handler ---
  const analyzeImageWithGemini = async (base64Image: string) => {
    try {
      // 1. Prepare the payload
      const prompt = `
        You are a strict and professional Architecture & Interior Design Professor. 
        Analyze the attached portfolio page image critically.
        
        Return the response in strictly valid JSON format without any markdown code blocks.
        The JSON structure must be:
        {
          "score": <integer 0-100>,
          "metrics": {
            "structure": <integer 0-100, logic flow>,
            "storytelling": <integer 0-100>,
            "visual": <integer 0-100, aesthetics>,
            "technical": <integer 0-100, drawing quality>,
            "creativity": <integer 0-100>
          },
          "critique": [
            { "type": "warning", "title": "<short title>", "desc": "<detailed critique in Korean>" },
            { "type": "good", "title": "<short title>", "desc": "<detailed praise in Korean>" },
            { "type": "suggestion", "title": "<short title>", "desc": "<specific improvement advice in Korean>" }
          ],
          "layoutFeedback": "<Detailed analysis of current layout and grid system in Korean>",
          "colorPalette": [
            { "hex": "<hex code>", "name": "<color name>" },
            { "hex": "<hex code>", "name": "<color name>" },
            { "hex": "<hex code>", "name": "<color name>" },
            { "hex": "<hex code>", "name": "<color name>" }
          ]
        }
      `;

      // Extract base64 data and mimeType dynamically
      // base64Image format: "data:image/png;base64,iVBORw0KGgo..."
      const matches = base64Image.match(/^data:(.+);base64,(.+)$/);
      if (!matches || matches.length !== 3) {
        throw new Error("Invalid image format");
      }
      const mimeType = matches[1];
      const base64Data = matches[2];

      // Use the correct model for the preview environment
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

      if (!response.ok) {
        const errorText = await response.text();
        console.error("Gemini API Error:", errorText);
        throw new Error(`API Request Failed: ${response.status} ${response.statusText}`);
      }

      const data = await response.json();
      const textResponse = data.candidates?.[0]?.content?.parts?.[0]?.text;
      
      if (!textResponse) {
        throw new Error("No response content from AI");
      }

      // Clean up markdown if present (handling ```json wrapper)
      const jsonString = textResponse.replace(/```json/g, '').replace(/```/g, '').trim();
      return JSON.parse(jsonString);

    } catch (err: any) {
      console.error(err);
      throw new Error(err.message || "AI 분석에 실패했습니다. 이미지를 다시 확인해주세요.");
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
      
      // Start Analysis
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

  const handleDragOver = (e: React.DragEvent) => { e.preventDefault(); };
  const handleDrop = (e: React.DragEvent) => {
    e.preventDefault();
    if (e.dataTransfer.files && e.dataTransfer.files[0]) {
      processFile(e.dataTransfer.files[0]);
    }
  };

  // --- Components ---

  const Header = () => (
    <header className="flex items-center justify-between px-6 py-4 bg-slate-900 text-white border-b border-slate-800">
      <div className="flex items-center space-x-2 cursor-pointer" onClick={() => setStep('upload')}>
        <Brain className="w-6 h-6 text-emerald-400" />
        <span className="text-xl font-bold tracking-tight">ArchFolio.AI <span className="text-xs bg-emerald-600 px-1 rounded ml-1">REAL-TIME</span></span>
      </div>
    </header>
  );

  const UploadSection = () => (
    <div className="flex flex-col items-center justify-center min-h-[80vh] bg-slate-50 p-6">
      <div className="text-center mb-10 max-w-2xl">
        <h1 className="text-4xl font-extrabold text-slate-900 mb-4">
          당신의 포트폴리오를 <span className="text-emerald-600">지금 바로</span> 분석합니다
        </h1>
        <p className="text-lg text-slate-600">
          페이지 이미지를 업로드하세요. Gemini Vision AI가<br />
          레이아웃, 논리, 디자인을 실시간으로 크리틱합니다.
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
          <p className="text-lg font-semibold text-slate-700">포트폴리오 이미지 업로드 (JPG, PNG)</p>
          <p className="text-sm text-slate-500 mt-2">브라우저 처리 속도를 위해 이미지를 권장합니다</p>
          <input type="file" className="hidden" accept="image/*" onChange={handleFileUpload} />
        </label>
        {error && (
          <div className="absolute bottom-4 text-red-500 text-sm flex items-center bg-red-50 px-3 py-1 rounded-full border border-red-200">
            <AlertTriangle className="w-4 h-4 mr-1" /> {error}
          </div>
        )}
      </div>
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
          <p className="text-slate-400 text-sm">구조, 텍스트 가독성, 디자인 요소를 스캔하고 있습니다.</p>
        </div>
      </div>
    </div>
  );

  const ResultDashboard = () => {
    if (!analysisResult) return null;

    return (
      <div className="min-h-screen bg-slate-50 pb-20">
        <div className="max-w-7xl mx-auto px-4 py-8">
          
          {/* Header Area */}
          <div className="flex flex-col md:flex-row justify-between items-start md:items-center mb-8 gap-4">
            <div>
              <div className="flex items-center gap-2 mb-1">
                <span className="bg-emerald-100 text-emerald-800 text-xs font-bold px-2 py-0.5 rounded">AI 분석 완료</span>
                <h2 className="text-2xl font-bold text-slate-900">진단 리포트</h2>
              </div>
              <p className="text-slate-500 text-sm">Target: {file?.name}</p>
            </div>
            <button 
              onClick={() => setStep('upload')}
              className="flex items-center px-4 py-2 border border-slate-300 bg-white text-slate-700 rounded-lg hover:bg-slate-50 transition shadow-sm text-sm"
            >
              <RefreshCw className="w-4 h-4 mr-2" /> 다른 이미지 분석하기
            </button>
          </div>

          <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
            
            {/* Left Column: Image & Score */}
            <div className="space-y-6">
              {/* Original Image */}
              <div className="bg-white p-4 rounded-xl shadow-sm border border-slate-100">
                <h3 className="text-sm font-semibold text-slate-500 mb-3 uppercase tracking-wide">Uploaded Page</h3>
                <img src={imagePreview!} alt="Original" className="w-full rounded border border-slate-200" />
              </div>

              {/* Score */}
              <div className="bg-white p-6 rounded-xl shadow-sm border border-slate-100">
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
                  {Object.entries(analysisResult.metrics).map(([key, value]: [string, any]) => (
                    <div key={key} className="flex items-center justify-between text-sm">
                      <span className="capitalize text-slate-600 w-24">{key}</span>
                      <div className="flex-1 h-2 bg-slate-100 rounded-full mx-3">
                        <div 
                          className={`h-full rounded-full ${Number(value) > 70 ? 'bg-emerald-500' : 'bg-slate-400'}`} 
                          style={{ width: `${value}%` }} 
                        />
                      </div>
                      <span className="font-bold text-slate-700">{value}</span>
                    </div>
                  ))}
                </div>
              </div>

               {/* Palette */}
               <div className="bg-white p-6 rounded-xl shadow-sm border border-slate-100">
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
              <div className="bg-white p-6 rounded-xl shadow-sm border border-slate-100">
                <h3 className="text-lg font-semibold text-slate-800 mb-4 flex items-center">
                  <Brain className="w-5 h-5 mr-2 text-slate-500" /> AI 상세 피드백
                </h3>
                <div className="space-y-4">
                  {analysisResult.critique.map((item: any, idx: number) => (
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
                </div>
              </div>

              {/* Layout AI Text Analysis */}
              <div className="bg-slate-900 text-white p-6 rounded-xl shadow-lg">
                <div className="flex justify-between items-center mb-4">
                  <h3 className="text-lg font-semibold flex items-center text-white">
                    <Layout className="w-5 h-5 mr-2 text-emerald-400" /> 레이아웃 분석 결과
                  </h3>
                </div>
                <div className="p-4 bg-slate-800 rounded-lg border border-slate-700 text-slate-300 text-sm leading-7">
                  {analysisResult.layoutFeedback}
                </div>
                <div className="mt-4 flex gap-2">
                   <div className="flex-1 bg-slate-800 h-32 rounded flex items-center justify-center border border-dashed border-slate-600 text-xs text-slate-500">
                      AI 제안 레이아웃 (Coming Soon)
                   </div>
                   <div className="flex-1 bg-slate-800 h-32 rounded flex items-center justify-center border border-dashed border-slate-600 text-xs text-slate-500">
                      레퍼런스 이미지 매칭 (Coming Soon)
                   </div>
                </div>
              </div>

            </div>
          </div>
        </div>
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
        @keyframes scan {
          0% { top: 0%; opacity: 0; }
          20% { opacity: 1; }
          80% { opacity: 1; }
          100% { top: 100%; opacity: 0; }
        }
        .animate-scan {
          animation: scan 2s cubic-bezier(0.4, 0, 0.2, 1) infinite;
        }
      `}</style>
    </div>
  );
}
