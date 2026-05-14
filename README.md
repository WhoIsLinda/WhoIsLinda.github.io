import React, { useState, useEffect, useMemo, useCallback } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken } from 'firebase/auth';
import { getFirestore, doc, setDoc, onSnapshot } from 'firebase/firestore';
import { 
  Trash2, Move, Settings2, ZoomIn, ZoomOut, ChevronDown, ChevronUp,
  FileText, Hand, MousePointer2, Tv, Tablet, Video, 
  Type, Eye, EyeOff, Palette, Lock, Unlock, FileDown, Link2
} from 'lucide-react';

// --- CONFIGURACIÓN FIREBASE ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'curaduria-pro-v3';

const App = () => {
  const [user, setUser] = useState(null);
  const [elements, setElements] = useState([]); 
  const [selectedId, setSelectedId] = useState(null);
  const [tool, setTool] = useState('select'); 
  const [zoom, setZoom] = useState(0.8);
  const [offset, setOffset] = useState({ x: 100, y: 100 });
  const [bgImage, setBgImage] = useState("");
  const [bgOpacity, setBgOpacity] = useState(0.5);
  const [bgSizeScale, setBgSizeScale] = useState(1);
  const [uiVisible, setUiVisible] = useState(true);
  const [dragging, setDragging] = useState(null);
  const [isPanning, setIsPanning] = useState(false);
  const [openSections, setOpenSections] = useState({ background: false, artwork: true, properties: true });
  
  const [newArt, setNewArt] = useState({ url: '', widthCm: 100, heightCm: 120, label: 'Nueva Obra' });

  // --- REGLA 3: AUTENTICACIÓN ---
  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (err) {
        console.error("Error de Autenticación:", err);
      }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, (currentUser) => {
      setUser(currentUser);
    });
    return () => unsubscribe();
  }, []);

  // --- REGLA 1: SINCRONIZACIÓN FIRESTORE (RUTA CORREGIDA A 6 SEGMENTOS) ---
  useEffect(() => {
    if (!user) return;

    // FIX: Document references must have an even number of segments.
    // artifacts (1) / appId (2) / public (3) / data (4) / main_collection (5) / state (6)
    const docRef = doc(db, 'artifacts', appId, 'public', 'data', 'main_collection', 'state');
    
    const unsubscribe = onSnapshot(docRef, 
      (docSnap) => {
        if (docSnap.exists()) {
          const data = docSnap.data();
          if (data.elements) setElements(data.elements);
          if (data.bgImage !== undefined) setBgImage(data.bgImage);
          if (data.bgOpacity !== undefined) setBgOpacity(data.bgOpacity);
          if (data.bgSizeScale !== undefined) setBgSizeScale(data.bgSizeScale);
        }
      }, 
      (error) => {
        console.error("Error de sincronización Firestore:", error);
      }
    );

    return () => unsubscribe();
  }, [user]);

  const saveToCloud = useCallback(async (newElements, extraData = {}) => {
    if (!user) return;
    const docRef = doc(db, 'artifacts', appId, 'public', 'data', 'main_collection', 'state');
    
    try {
      const dataToSave = {
        elements: newElements || elements,
        bgImage: extraData.bgImage !== undefined ? extraData.bgImage : bgImage,
        bgOpacity: extraData.bgOpacity !== undefined ? extraData.bgOpacity : bgOpacity,
        bgSizeScale: extraData.bgSizeScale !== undefined ? extraData.bgSizeScale : bgSizeScale,
        lastUpdated: Date.now()
      };
      await setDoc(docRef, dataToSave, { merge: true });
    } catch (err) {
      console.error("Error al guardar en la nube:", err);
    }
  }, [user, elements, bgImage, bgOpacity, bgSizeScale]);

  // --- ATAJOS DE TECLADO ---
  useEffect(() => {
    const handleKeyDown = (e) => {
      if (e.target.tagName === 'INPUT' || e.target.tagName === 'TEXTAREA') return;
      
      const key = e.key.toLowerCase();
      if (key === 'v') setTool('select');
      else if (key === 'h') setTool('hand');
      else if (key === 'delete' || key === 'backspace') {
        if (selectedId) {
          const updated = elements.filter(el => el.id !== selectedId);
          setElements(updated);
          setSelectedId(null);
          saveToCloud(updated);
        }
      }
    };
    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, [selectedId, elements, saveToCloud]);

  // --- GESTIÓN DE ELEMENTOS ---
  const addElement = (type) => {
    const id = `${type}-${Date.now()}`;
    const defaultProps = {
      id, type,
      x: (400 - offset.x) / zoom,
      y: (400 - offset.y) / zoom,
      rotation: 0, opacity: 1, locked: false
    };

    let specific = {};
    switch(type) {
      case 'wall': specific = { width: 200, thickness: 15, color: '#334155', label: 'Muro' }; break;
      case 'title': specific = { width: 300, height: 60, label: 'TÍTULO EXPO', color: '#0f172a', fontSize: 32 }; break;
      case 'text': specific = { width: 120, height: 160, label: 'Texto Curatorial', color: '#1e293b', fontSize: 14, bgColor: '#ffffff' }; break;
      case 'screen': specific = { width: 120, height: 70, label: 'Pantalla', color: '#000000' }; break;
      case 'ipad': specific = { width: 30, height: 20, label: 'iPad', color: '#1e293b' }; break;
      case 'projector': specific = { width: 40, height: 40, label: 'Proyector', color: '#475569' }; break;
      case 'zone': specific = { width: 400, height: 400, label: 'Zona de Color', color: '#e2e8f0', opacity: 0.5, locked: true }; break;
      case 'area_text': specific = { width: 250, height: 100, label: 'Cédula de Obra', color: '#475569', fontSize: 12, bgColor: '#f8fafc' }; break;
      default: break;
    }

    const updated = [...elements, { ...defaultProps, ...specific }];
    setElements(updated);
    setSelectedId(id);
    saveToCloud(updated);
  };

  const addArtwork = () => {
    if (!newArt.url) return;
    const artId = `art-${Date.now()}`;
    const updated = [...elements, {
      id: artId, type: 'art', ...newArt,
      x: (500 - offset.x) / zoom, y: (500 - offset.y) / zoom,
      rotation: 0, opacity: 1, locked: false
    }];
    setElements(updated);
    setSelectedId(artId);
    setNewArt({ url: '', widthCm: 100, heightCm: 120, label: 'Nueva Obra' });
    saveToCloud(updated);
  };

  // --- LÓGICA DE INTERACCIÓN ---
  const handleMouseDown = (e, id) => {
    if (tool === 'hand') {
      setIsPanning(true);
      setDragging({ startX: e.clientX, startY: e.clientY, origOffsetX: offset.x, origOffsetY: offset.y });
      return;
    }
    if (id) {
      e.stopPropagation();
      const item = elements.find(el => el.id === id);
      if (item.locked && tool !== 'select') return;
      setSelectedId(id);
      if (!item.locked) {
        setDragging({ id, startX: e.clientX, startY: e.clientY, origX: item.x, origY: item.y });
      }
    } else {
      setSelectedId(null);
    }
  };

  const handleMouseMove = (e) => {
    if (!dragging) return;
    if (isPanning) {
      setOffset({ x: dragging.origOffsetX + (e.clientX - dragging.startX), y: dragging.origOffsetY + (e.clientY - dragging.startY) });
      return;
    }
    const dx = (e.clientX - dragging.startX) / zoom;
    const dy = (e.clientY - dragging.startY) / zoom;
    setElements(prev => prev.map(el => el.id === dragging.id ? { ...el, x: dragging.origX + dx, y: dragging.origY + dy } : el));
  };

  const handleMouseUp = () => {
    if (dragging && !isPanning) saveToCloud(elements);
    setDragging(null);
    setIsPanning(false);
  };

  const updateSelected = (field, value) => {
    const updated = elements.map(el => el.id === selectedId ? { ...el, [field]: value } : el);
    setElements(updated);
    saveToCloud(updated);
  };

  const selectedItem = useMemo(() => elements.find(el => el.id === selectedId), [elements, selectedId]);

  const stats = useMemo(() => ({
    obras: elements.filter(e => e.type === 'art').length,
    muros: elements.filter(e => e.type === 'wall').length,
    textos: elements.filter(e => ['text', 'area_text', 'title'].includes(e.type)).length
  }), [elements]);

  return (
    <div className={`flex h-screen w-full bg-slate-100 overflow-hidden font-sans select-none ${tool === 'hand' ? 'cursor-grab active:cursor-grabbing' : ''}`}>
      
      {uiVisible && (
        <aside className="w-80 bg-white border-r shadow-xl z-50 flex flex-col no-print">
          {/* Header */}
          <div className="p-4 bg-slate-900 text-white flex items-center justify-between">
            <div className="flex items-center gap-2">
              <Settings2 size={16} className="text-indigo-400" />
              <span className="font-black text-xs uppercase tracking-tighter">Panel de Curaduría</span>
            </div>
            <div className="flex flex-col items-end gap-1">
              <div className="flex gap-1 bg-slate-800 p-1 rounded-lg">
                <button 
                  onClick={() => setTool('select')} 
                  className={`p-1.5 rounded transition-colors ${tool === 'select' ? 'bg-indigo-600 text-white' : 'text-slate-400 hover:text-white'}`}
                >
                  <MousePointer2 size={14} />
                </button>
                <button 
                  onClick={() => setTool('hand')} 
                  className={`p-1.5 rounded transition-colors ${tool === 'hand' ? 'bg-indigo-600 text-white' : 'text-slate-400 hover:text-white'}`}
                >
                  <Hand size={14} />
                </button>
              </div>
            </div>
          </div>

          {/* Estadísticas */}
          <div className="px-4 py-3 bg-white border-b grid grid-cols-3 gap-2">
            <StatBox label="OBRAS" val={stats.obras} />
            <StatBox label="MUROS" val={stats.muros} />
            <StatBox label="TEXTOS" val={stats.textos} />
          </div>

          {/* Grid de herramientas rápidas */}
          <div className="p-4 bg-white border-b">
            <label className="text-[10px] font-bold text-slate-400 block mb-3 uppercase tracking-widest">Añadir Elementos</label>
            <div className="grid grid-cols-4 gap-2">
              <ToolBtn icon={Move} label="Muro" onClick={() => addElement('wall')} />
              <ToolBtn icon={Palette} label="Zona" onClick={() => addElement('zone')} color="text-amber-500" />
              <ToolBtn icon={Type} label="Título" onClick={() => addElement('title')} />
              <ToolBtn icon={FileText} label="Texto" onClick={() => addElement('text')} />
              <ToolBtn icon={Tv} label="Monitor" onClick={() => addElement('screen')} />
              <ToolBtn icon={Tablet} label="iPad" onClick={() => addElement('ipad')} />
              <ToolBtn icon={Video} label="Proy." onClick={() => addElement('projector')} />
              <ToolBtn icon={Link2} label="Cédula" onClick={() => addElement('area_text')} />
            </div>
          </div>

          <div className="flex-1 overflow-y-auto bg-slate-50/50">
            <Section 
              title="Añadir Obra Nueva" 
              isOpen={openSections.artwork} 
              onToggle={() => setOpenSections(prev => ({...prev, artwork: !prev.artwork}))}
            >
              <div className="space-y-3">
                <input type="text" placeholder="URL de imagen..." className="w-full border rounded p-2 text-xs focus:ring-1 focus:ring-indigo-500 outline-none" value={newArt.url} onChange={e => setNewArt({...newArt, url: e.target.value})} />
                <input type="text" placeholder="Nombre de la pieza" className="w-full border rounded p-2 text-xs focus:ring-1 focus:ring-indigo-500 outline-none" value={newArt.label} onChange={e => setNewArt({...newArt, label: e.target.value})} />
                <button onClick={addArtwork} className="w-full bg-indigo-600 text-white font-bold p-2 rounded text-xs hover:bg-indigo-700 transition-colors shadow-sm">Cargar al Plano</button>
              </div>
            </Section>

            {selectedItem && (
              <Section 
                title={`Editar: ${selectedItem.type.toUpperCase()}`} 
                isOpen={openSections.properties} 
                onToggle={() => setOpenSections(prev => ({...prev, properties: !prev.properties}))}
                highlight
              >
                <div className="space-y-4">
                  <div className="flex justify-between items-center">
                    <button onClick={() => updateSelected('locked', !selectedItem.locked)} className={`flex items-center gap-1 text-[10px] font-bold px-2 py-1 rounded border transition-colors ${selectedItem.locked ? 'bg-amber-100 text-amber-700 border-amber-200' : 'bg-white text-slate-500 hover:bg-slate-50'}`}>
                      {selectedItem.locked ? <><Lock size={12}/> BLOQUEADO</> : <><Unlock size={12}/> DESBLOQUEADO</>}
                    </button>
                    <button 
                      onClick={() => { 
                        const upd = elements.filter(e => e.id !== selectedId); 
                        setElements(upd); 
                        setSelectedId(null); 
                        saveToCloud(upd); 
                      }} 
                      className="text-red-500 hover:bg-red-50 p-1.5 rounded transition-colors"
                    >
                      <Trash2 size={16}/>
                    </button>
                  </div>

                  <PropRow label={`Rotación: ${selectedItem.rotation}°`}>
                    <input type="range" min="0" max="360" value={selectedItem.rotation} onChange={e => updateSelected('rotation', parseInt(e.target.value))} className="w-full accent-indigo-600" />
                  </PropRow>

                  <PropRow label="Color / Opacidad">
                    <div className="flex gap-2 items-center">
                      <input type="color" value={selectedItem.color || selectedItem.bgColor || '#000000'} onChange={e => updateSelected(selectedItem.color ? 'color' : 'bgColor', e.target.value)} className="h-8 w-10 bg-transparent cursor-pointer" />
                      <input type="range" min="0.1" max="1" step="0.1" value={selectedItem.opacity} onChange={e => updateSelected('opacity', parseFloat(e.target.value))} className="flex-1 accent-indigo-600" />
                    </div>
                  </PropRow>

                  <div className="grid grid-cols-2 gap-2">
                    <PropRow label="Ancho (cm)"><input type="number" value={selectedItem.width || selectedItem.widthCm} onChange={e => updateSelected(selectedItem.width ? 'width' : 'widthCm', parseFloat(e.target.value))} className="w-full border p-1 rounded text-xs focus:ring-1 focus:ring-indigo-500 outline-none" /></PropRow>
                    <PropRow label="Alto (cm)"><input type="number" value={selectedItem.height || selectedItem.heightCm || selectedItem.thickness} onChange={e => updateSelected(selectedItem.height ? 'height' : (selectedItem.heightCm ? 'heightCm' : 'thickness'), parseFloat(e.target.value))} className="w-full border p-1 rounded text-xs focus:ring-1 focus:ring-indigo-500 outline-none" /></PropRow>
                  </div>

                  <PropRow label="Texto Visible">
                    <textarea value={selectedItem.label} onChange={e => updateSelected('label', e.target.value)} className="w-full border rounded p-2 text-xs h-20 focus:ring-1 focus:ring-indigo-500 outline-none resize-none" />
                  </PropRow>
                </div>
              </Section>
            )}

            <Section 
              title="Plano de Referencia" 
              isOpen={openSections.background} 
              onToggle={() => setOpenSections(prev => ({...prev, background: !prev.background}))}
            >
              <div className="space-y-3">
                <input type="text" placeholder="URL del plano..." className="w-full border rounded p-2 text-xs focus:ring-1 focus:ring-indigo-500 outline-none" value={bgImage} onChange={e => {setBgImage(e.target.value); saveToCloud(elements, {bgImage: e.target.value});}} />
                <div className="grid grid-cols-2 gap-2">
                  <PropRow label="Opacidad"><input type="range" min="0" max="1" step="0.1" value={bgOpacity} onChange={e => {setBgOpacity(parseFloat(e.target.value)); saveToCloud(elements, {bgOpacity: parseFloat(e.target.value)});}} className="w-full accent-indigo-600" /></PropRow>
                  <PropRow label="Escala"><input type="range" min="0.1" max="5" step="0.1" value={bgSizeScale} onChange={e => {setBgSizeScale(parseFloat(e.target.value)); saveToCloud(elements, {bgSizeScale: parseFloat(e.target.value)});}} className="w-full accent-indigo-600" /></PropRow>
                </div>
              </div>
            </Section>
          </div>

          <div className="p-4 border-t bg-white space-y-2">
            <button onClick={() => window.print()} className="w-full flex items-center justify-center gap-2 bg-slate-800 text-white p-2.5 rounded-xl text-xs font-bold hover:bg-black transition-all shadow-sm active:scale-95">
              <FileDown size={14}/> Exportar Plano (PDF)
            </button>
            <div className="text-[8px] text-slate-300 font-mono text-center uppercase tracking-widest">
              ID: {appId}
            </div>
          </div>
        </aside>
      )}

      {/* Canvas Principal */}
      <main className="flex-1 relative overflow-hidden bg-slate-200" onMouseMove={handleMouseMove} onMouseUp={handleMouseUp} onMouseDown={(e) => tool === 'hand' && handleMouseDown(e)}>
        <div className="absolute top-6 right-6 z-40 flex flex-col gap-2 no-print">
          <button onClick={() => setUiVisible(!uiVisible)} className="bg-white p-3 rounded-full shadow-lg border hover:scale-110 transition-all text-slate-600 hover:text-indigo-600">
            {uiVisible ? <EyeOff size={20}/> : <Eye size={20}/>}
          </button>
          <div className="bg-white rounded-2xl shadow-xl border p-1 flex flex-col items-center">
            <button onClick={() => setZoom(z => Math.min(4, z + 0.1))} className="p-2 hover:bg-slate-100 rounded-xl text-slate-600 transition-colors"><ZoomIn size={18}/></button>
            <span className="py-1 font-bold text-[10px] text-slate-400">{Math.round(zoom * 100)}%</span>
            <button onClick={() => setZoom(z => Math.max(0.1, z - 0.1))} className="p-2 hover:bg-slate-100 rounded-xl text-slate-600 transition-colors"><ZoomOut size={18}/></button>
          </div>
        </div>

        <div 
          className="w-[10000px] h-[10000px] absolute origin-top-left transition-none"
          style={{ 
            transform: `translate(${offset.x}px, ${offset.y}px) scale(${zoom})`,
            backgroundImage: 'radial-gradient(#cbd5e1 1px, transparent 1px)',
            backgroundSize: '40px 40px'
          }}
        >
          {bgImage && (
            <img 
              src={bgImage} 
              className="absolute top-0 left-0 pointer-events-none origin-top-left" 
              style={{ opacity: bgOpacity, transform: `scale(${bgSizeScale})`, maxWidth: 'none' }} 
              alt="Plano Base" 
              onError={(e) => { e.target.style.display = 'none'; }}
            />
          )}

          {elements.map(el => {
            const isSelected = selectedId === el.id;
            const w = (el.width || el.widthCm) / 2;
            const h = (el.height || el.heightCm || el.thickness) / 2;
            
            let zIndex = "z-10";
            if (el.type === 'zone') zIndex = "z-0";
            if (isSelected) zIndex = "z-50";

            return (
              <div 
                key={el.id}
                onMouseDown={(e) => handleMouseDown(e, el.id)}
                className={`absolute flex items-center justify-center transition-shadow ${zIndex} 
                  ${isSelected ? 'ring-4 ring-indigo-500 shadow-2xl scale-[1.01]' : 'shadow-sm'} 
                  ${el.locked ? 'cursor-default' : 'cursor-move'}`}
                style={{
                  left: el.x, top: el.y, width: w, height: h,
                  transform: `rotate(${el.rotation}deg)`,
                  backgroundColor: el.bgColor || el.color || 'transparent',
                  opacity: el.opacity ?? 1,
                  border: el.type === 'wall' ? '1px solid rgba(0,0,0,0.2)' : (el.type === 'zone' ? 'none' : '1px solid rgba(0,0,0,0.05)'),
                  borderRadius: el.type === 'ipad' ? '4px' : '0px'
                }}
              >
                {el.type === 'art' && <img src={el.url} className="w-full h-full object-cover pointer-events-none" alt={el.label} onError={(e) => { e.target.src = 'https://via.placeholder.com/150?text=Error+Imagen'; }} />}
                {el.type === 'title' && <div className="font-bold text-center leading-none px-2 uppercase" style={{ fontSize: `${el.fontSize / 2}px`, color: el.color }}>{el.label}</div>}
                {el.type === 'text' && <div className="p-2 w-full h-full overflow-hidden leading-tight text-[8px]" style={{ color: el.color }}>{el.label}</div>}
                {el.type === 'area_text' && <div className="p-2 w-full h-full flex flex-col justify-center border-l-4 border-indigo-500" style={{ fontSize: `${el.fontSize / 2}px`, color: el.color }}>{el.label}</div>}
                {el.type === 'screen' && <Tv size={Math.min(w, h) * 0.5} className="text-white/20" />}
                {el.type === 'ipad' && <Tablet size={Math.min(w, h) * 0.5} className="text-white/20" />}
                {el.type === 'projector' && <Video size={Math.min(w, h) * 0.5} className="text-white/20" />}
                {el.type === 'zone' && <div className="text-[9px] font-black text-black/10 text-center uppercase pointer-events-none">{el.label}</div>}
                
                {isSelected && (
                  <div className="absolute -top-8 left-0 flex gap-2 no-print">
                    <div className="bg-slate-900 text-white text-[9px] px-2 py-0.5 rounded font-bold whitespace-nowrap uppercase tracking-tighter shadow-xl">
                      {el.label.substring(0, 15)}... {el.locked && '(🔒)'}
                    </div>
                  </div>
                )}
              </div>
            );
          })}
        </div>
      </main>

      <style>{`
        @media print {
          .no-print { display: none !important; }
          body, html { background: white !important; }
          main { background: white !important; width: 100% !important; height: 100% !important; position: relative !important; top:0 !important; left:0 !important; }
        }
      `}</style>
    </div>
  );
};

// --- COMPONENTES AUXILIARES ---
const StatBox = ({ label, val }) => (
  <div className="text-center border rounded-lg py-1 bg-slate-50">
    <div className="text-[9px] font-bold text-slate-400">{label}</div>
    <div className="text-sm font-black text-slate-800">{val}</div>
  </div>
);

// FIX: Pasamos el componente de icono (función) en lugar del elemento JSX renderizado para evitar errores de React
const ToolBtn = ({ icon: Icon, label, onClick, color = "text-slate-600" }) => (
  <button onClick={onClick} className="flex flex-col items-center gap-1 p-2 border rounded-xl hover:bg-slate-50 transition-all active:scale-95 group">
    <div className={`${color} group-hover:scale-110 transition-transform`}>
      <Icon size={18} />
    </div>
    <span className="text-[8px] font-bold text-slate-400 uppercase tracking-tighter">{label}</span>
  </button>
);

const Section = ({ title, children, isOpen, onToggle, highlight = false }) => (
  <div className={`border-b border-slate-200 transition-colors ${isOpen ? 'bg-white' : 'bg-transparent'}`}>
    <button onClick={onToggle} className={`w-full flex items-center justify-between p-4 hover:bg-white transition-colors ${highlight && isOpen ? 'border-l-4 border-indigo-600' : ''}`}>
      <span className={`text-[10px] font-black uppercase tracking-widest ${highlight ? 'text-indigo-600' : 'text-slate-600'}`}>{title}</span>
      {isOpen ? <ChevronUp size={14} className="text-slate-400" /> : <ChevronDown size={14} className="text-slate-400" />}
    </button>
    {isOpen && <div className="px-4 pb-6 animate-in slide-in-from-top-2 duration-200">{children}</div>}
  </div>
);

const PropRow = ({ label, children }) => (
  <div className="space-y-1">
    <label className="text-[9px] font-bold text-slate-400 uppercase tracking-tighter">{label}</label>
    {children}
  </div>
);
