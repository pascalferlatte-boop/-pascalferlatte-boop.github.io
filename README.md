# -pascalferlatte-boop.github.io
import React, { useState, useEffect, useCallback } from 'react';
import { 
  MapPin, 
  Camera, 
  Barcode, 
  Navigation, 
  Trash2, 
  CheckCircle, 
  Plus, 
  Play, 
  X,
  Maximize,
  ArrowRight,
  Search,
  Loader2,
  Home,
  ArrowRightCircle,
  Flag,
  ChevronRight,
  ShieldCheck,
  Car,
  GripVertical,
  Minimize2
} from 'lucide-react';

// Configuration de l'API (Clé vide par défaut)
const apiKey = ""; 

const App = () => {
  const [addresses, setAddresses] = useState([]);
  const [inputValue, setInputValue] = useState('');
  const [suggestions, setSuggestions] = useState([]);
  const [userLocation, setUserLocation] = useState(null);
  const [isOptimizing, setIsOptimizing] = useState(false);
  const [currentView, setCurrentView] = useState('list');
  const [isCapturing, setIsCapturing] = useState(false);
  const [isSearching, setIsSearching] = useState(false);
  const [routeType, setRouteType] = useState('one-way');
  const [showInAppNav, setShowInAppNav] = useState(false);
  
  // État pour gérer la livraison active
  const [activeDeliveryIndex, setActiveDeliveryIndex] = useState(null);

  // État pour le Drag & Drop manuel
  const [draggedItemIndex, setDraggedItemIndex] = useState(null);

  useEffect(() => {
    const saved = localStorage.getItem('delivery_addresses');
    if (saved) setAddresses(JSON.parse(saved));

    if (navigator.geolocation) {
      navigator.geolocation.getCurrentPosition(
        (pos) => setUserLocation({ lat: pos.coords.latitude, lng: pos.coords.longitude }),
        (err) => console.log("Géolocalisation non disponible")
      );
    }
  }, []);

  useEffect(() => {
    localStorage.setItem('delivery_addresses', JSON.stringify(addresses));
  }, [addresses]);

  // --- LOGIQUE AUTO-COMPLÉTION ---

  const fetchSuggestions = useCallback(async (text) => {
    if (text.length < 3 || !apiKey) {
      setSuggestions([]);
      return;
    }

    setIsSearching(true);
    try {
      let url = `https://maps.googleapis.com/maps/api/place/autocomplete/json?input=${encodeURIComponent(text)}&key=${apiKey}&types=address&language=fr`;
      if (userLocation) {
        url += `&location=${userLocation.lat},${userLocation.lng}&radius=50000`;
      }
      const response = await fetch(url);
      const data = await response.json();
      if (data.predictions) {
        setSuggestions(data.predictions.map(p => p.description));
      }
    } catch (error) {
      console.error("Erreur suggestions:", error);
    } finally {
      setIsSearching(false);
    }
  }, [userLocation]);

  useEffect(() => {
    const timer = setTimeout(() => {
      fetchSuggestions(inputValue);
    }, 300);
    return () => clearTimeout(timer);
  }, [inputValue, fetchSuggestions]);

  // --- ACTIONS ---

  const addAddress = (text) => {
    if (!text.trim()) return;
    const newAddr = {
      id: crypto.randomUUID(),
      text: text.trim(),
      completed: false,
      timestamp: new Date().toISOString()
    };
    setAddresses([...addresses, newAddr]);
    setInputValue('');
    setSuggestions([]);
  };

  const optimizeRoute = () => {
    if (addresses.length < 1) return;
    setIsOptimizing(true);

    setTimeout(() => {
      let unvisited = [...addresses.filter(a => !a.completed)];
      let completed = [...addresses.filter(a => a.completed)];
      
      unvisited.sort((a, b) => a.text.length - b.text.length);

      const newOrder = [...completed, ...unvisited];
      setAddresses(newOrder);
      setIsOptimizing(false);
      
      const firstPending = newOrder.findIndex(a => !a.completed);
      setActiveDeliveryIndex(firstPending !== -1 ? firstPending : null);
      setCurrentView('map');
    }, 1200);
  };

  const confirmDelivery = () => {
    if (activeDeliveryIndex === null) return;
    
    const updated = [...addresses];
    updated[activeDeliveryIndex].completed = true;
    setAddresses(updated);

    const nextIndex = updated.findIndex((a, idx) => !a.completed && idx > activeDeliveryIndex);
    const anyPending = updated.findIndex(a => !a.completed);

    setShowInAppNav(false); // Fermer la navigation interne après confirmation

    if (nextIndex !== -1) {
      setActiveDeliveryIndex(nextIndex);
    } else if (anyPending !== -1) {
      setActiveDeliveryIndex(anyPending);
    } else {
      setActiveDeliveryIndex(null);
    }
  };

  const removeAddress = (id) => {
    setAddresses(addresses.filter(a => a.id !== id));
  };

  // --- DRAG & DROP ---

  const onDragStart = (index) => setDraggedItemIndex(index);

  const onDragOver = (e, index) => {
    e.preventDefault();
    if (draggedItemIndex === null || draggedItemIndex === index) return;
    const newAddresses = [...addresses];
    const item = newAddresses.splice(draggedItemIndex, 1)[0];
    newAddresses.splice(index, 0, item);
    setDraggedItemIndex(index);
    setAddresses(newAddresses);
  };

  const onDragEnd = () => {
    setDraggedItemIndex(null);
    const firstPending = addresses.findIndex(a => !a.completed);
    setActiveDeliveryIndex(firstPending !== -1 ? firstPending : null);
  };

  // --- UI COMPONENTS ---

  const NavigationIframe = ({ destination }) => {
    // Mode intégré de Google Maps (nécessite une clé API pour un fonctionnement optimal, 
    // ici on utilise le mode d'aperçu de recherche pour la démo si apiKey est vide)
    const encodedDest = encodeURIComponent(destination);
    const mapUrl = apiKey 
      ? `https://www.google.com/maps/embed/v1/directions?key=${apiKey}&destination=${encodedDest}&mode=driving`
      : `https://www.google.com/maps?q=${encodedDest}&output=embed&z=15`;

    return (
      <div className="absolute inset-0 z-50 bg-white flex flex-col animate-in slide-in-from-right duration-300">
        <div className="bg-slate-900 text-white p-4 flex justify-between items-center shadow-lg">
          <div className="flex items-center gap-2">
            <Car size={18} className="text-blue-400" />
            <span className="text-xs font-bold uppercase truncate max-w-[200px]">{destination}</span>
          </div>
          <button 
            onClick={() => setShowInAppNav(false)} 
            className="p-2 bg-white/10 rounded-full hover:bg-white/20 transition-colors"
          >
            <Minimize2 size={20} />
          </button>
        </div>
        <div className="flex-1 relative">
          <iframe
            title="Internal Navigation"
            width="100%"
            height="100%"
            frameBorder="0"
            src={mapUrl}
            allowFullScreen
          ></iframe>
        </div>
        <div className="p-4 bg-white border-t border-slate-200 grid grid-cols-2 gap-3">
          <button 
            onClick={() => setShowInAppNav(false)} 
            className="py-3 px-4 border border-slate-200 rounded-xl text-slate-600 font-bold text-sm"
          >
            Quitter
          </button>
          <button 
            onClick={confirmDelivery}
            className="py-3 px-4 bg-green-600 text-white rounded-xl font-bold text-sm shadow-lg shadow-green-100"
          >
            Marquer Livré
          </button>
        </div>
      </div>
    );
  };

  const currentTask = activeDeliveryIndex !== null ? addresses[activeDeliveryIndex] : null;

  return (
    <div className="min-h-screen bg-slate-50 font-sans pb-32 overflow-x-hidden">
      <div className="bg-blue-700 text-white p-4 shadow-lg sticky top-0 z-30 flex justify-between items-center">
        <h1 className="text-xl font-bold flex items-center gap-2">
          <Navigation size={22} className="fill-current text-blue-200" /> Livreur Pro
        </h1>
        <div className="flex bg-blue-800 rounded-lg p-1">
          <button onClick={() => setCurrentView('list')} className={`px-4 py-1.5 rounded-md text-sm font-medium transition-colors ${currentView === 'list' ? 'bg-white text-blue-700 shadow' : 'text-blue-200'}`}>Liste</button>
          <button onClick={() => setCurrentView('map')} className={`px-4 py-1.5 rounded-md text-sm font-medium transition-colors ${currentView === 'map' ? 'bg-white text-blue-700 shadow' : 'text-blue-200'}`}>Carte</button>
        </div>
      </div>

      {addresses.length > 0 && (
        <div className="bg-white px-4 py-3 border-b border-slate-100 shadow-sm">
          <div className="flex justify-between items-end mb-2">
            <div className="flex items-center gap-1.5">
              <ShieldCheck size={14} className="text-green-600" />
              <span className="text-[10px] font-bold text-slate-400 uppercase tracking-widest">Navigation Interne Activée</span>
            </div>
            <span className="text-sm font-black text-blue-700">
              {addresses.filter(a => a.completed).length} / {addresses.length}
            </span>
          </div>
          <div className="h-1.5 w-full bg-slate-100 rounded-full overflow-hidden">
            <div className="h-full bg-blue-600 transition-all duration-500" style={{ width: `${(addresses.filter(a => a.completed).length / addresses.length) * 100}%` }} />
          </div>
        </div>
      )}

      <main className="max-w-md mx-auto p-4 space-y-6">
        {currentView === 'list' && (
          <div className="space-y-4">
            <div className="bg-white p-4 rounded-2xl shadow-sm border border-slate-200 space-y-3">
              <div className="flex gap-2">
                <div className="flex-1 relative">
                  <input 
                    type="text" value={inputValue} onChange={(e) => setInputValue(e.target.value)}
                    placeholder="Adresse du colis..." className="w-full p-3 bg-slate-50 border border-slate-200 rounded-xl focus:outline-none focus:ring-2 focus:ring-blue-500"
                  />
                  <div className="absolute right-3 top-3.5 text-slate-400">
                    {isSearching ? <Loader2 size={18} className="animate-spin" /> : <Search size={18} />}
                  </div>
                </div>
                <button onClick={() => addAddress(inputValue)} className="bg-blue-600 text-white p-3 rounded-xl shadow-md"><Plus size={20} /></button>
              </div>
              <div className="grid grid-cols-2 gap-3 pt-1">
                <button onClick={() => setIsCapturing(true)} className="flex items-center justify-center gap-2 bg-slate-100 p-3 rounded-xl text-slate-700 text-sm font-semibold hover:bg-slate-200"><Camera size={18} /> Photo</button>
                <button onClick={() => addAddress("12 Rue de la Paix, Paris")} className="flex items-center justify-center gap-2 bg-slate-100 p-3 rounded-xl text-slate-700 text-sm font-semibold"><Barcode size={18} /> Scanner</button>
              </div>
            </div>

            <div className="space-y-3">
              <h2 className="text-slate-500 text-xs font-bold uppercase tracking-wider px-1">Tournée du jour</h2>
              {addresses.map((addr, index) => (
                <div 
                  key={addr.id} draggable={!addr.completed} onDragStart={() => onDragStart(index)} onDragOver={(e) => onDragOver(e, index)} onDragEnd={onDragEnd}
                  className={`bg-white p-4 rounded-2xl border border-slate-200 shadow-sm flex items-center gap-3 transition-all ${addr.completed ? 'bg-slate-50 opacity-60' : 'cursor-grab hover:border-blue-200'} ${draggedItemIndex === index ? 'opacity-30 border-blue-500' : ''}`}
                >
                  {!addr.completed && <GripVertical size={18} className="text-slate-300" />}
                  <div className={`w-8 h-8 rounded-full flex items-center justify-center font-bold text-xs ${index === activeDeliveryIndex ? 'bg-blue-600 text-white ring-4 ring-blue-100' : addr.completed ? 'bg-green-100 text-green-600' : 'bg-slate-100 text-slate-500'}`}>
                    {addr.completed ? <CheckCircle size={14} /> : index + 1}
                  </div>
                  <p className={`flex-1 text-sm font-semibold truncate ${addr.completed ? 'line-through text-slate-400' : 'text-slate-800'}`}>{addr.text}</p>
                  <div className="flex gap-1">
                    <button onClick={() => { setActiveDeliveryIndex(index); setCurrentView('map'); }} className="p-2 text-blue-600 hover:bg-blue-50 rounded-full"><ChevronRight size={18} /></button>
                    <button onClick={() => removeAddress(addr.id)} className="p-2 text-slate-300 hover:text-red-500"><Trash2 size={18} /></button>
                  </div>
                </div>
              ))}
            </div>
          </div>
        )}

        {currentView === 'map' && (
          <div className="space-y-4">
            <div className="bg-slate-200 aspect-[16/10] rounded-3xl flex items-center justify-center relative overflow-hidden shadow-inner border border-white">
               {showInAppNav && currentTask ? (
                 <NavigationIframe destination={currentTask.text} />
               ) : (
                 <div className="relative text-center p-6 bg-white/90 backdrop-blur-md rounded-2xl shadow-xl border border-white max-w-[85%]">
                   <Car className="text-blue-600 mx-auto mb-2" size={32} />
                   <p className="text-slate-800 font-black text-sm uppercase">Navigation Embarquée</p>
                   <p className="text-[10px] text-slate-500 mt-1 uppercase font-bold">Rester dans l'application</p>
                 </div>
               )}
            </div>

            {currentTask ? (
              <div className="bg-white p-6 rounded-3xl shadow-xl border border-slate-100">
                <div className="flex justify-between items-start mb-4">
                  <span className="bg-blue-100 text-blue-700 text-[10px] font-black px-2 py-1 rounded-md uppercase tracking-widest">Étape #{activeDeliveryIndex + 1}</span>
                </div>
                <h3 className="font-black text-slate-800 text-xl leading-tight mb-6">{currentTask.text}</h3>
                <div className="grid grid-cols-2 gap-3">
                  <button onClick={() => setShowInAppNav(true)} className="bg-blue-600 text-white py-4 rounded-2xl flex items-center justify-center gap-2 font-black transition-transform active:scale-95 shadow-lg shadow-blue-100">
                    <Play size={20} className="fill-current" /> NAVIGUER
                  </button>
                  <button onClick={confirmDelivery} className="bg-green-600 text-white py-4 rounded-2xl flex items-center justify-center gap-2 font-black shadow-lg shadow-green-100 transition-transform active:scale-95">
                    <CheckCircle size={20} /> LIVRÉ
                  </button>
                </div>
              </div>
            ) : (
              <div className="bg-green-600 text-white p-8 rounded-3xl shadow-xl text-center space-y-4">
                <Flag size={48} className="mx-auto" />
                <h3 className="font-black text-xl">Tournée Terminée !</h3>
                <button onClick={() => { setAddresses([]); setCurrentView('list'); }} className="w-full bg-white text-green-700 py-4 rounded-2xl font-black">RETOUR</button>
              </div>
            )}
          </div>
        )}
      </main>

      {currentView === 'list' && addresses.some(a => !a.completed) && (
        <div className="fixed bottom-0 left-0 right-0 p-4 bg-white/80 backdrop-blur-md border-t border-slate-100 flex justify-center max-w-md mx-auto z-40">
          <button onClick={optimizeRoute} disabled={isOptimizing} className="w-full bg-blue-700 text-white py-4 rounded-2xl font-black flex items-center justify-center gap-2 shadow-2xl disabled:bg-blue-300">
            {isOptimizing ? <Loader2 size={24} className="animate-spin" /> : <><Maximize size={20} /> OPTIMISER LA TOURNEE</>}
          </button>
        </div>
      )}

      {isCapturing && (
        <div className="fixed inset-0 bg-slate-900 z-50 flex flex-col p-6">
          <button onClick={() => setIsCapturing(false)} className="self-end bg-white/10 p-2 rounded-full text-white"><X /></button>
          <div className="flex-1 flex items-center justify-center">
             <div className="w-full aspect-[3/4] border-2 border-dashed border-blue-400/50 rounded-3xl flex flex-col items-center justify-center bg-slate-800 overflow-hidden relative">
               <div className="absolute inset-x-0 h-1 bg-blue-500 shadow-[0_0_15px_rgba(59,130,246,0.8)] animate-[scan_2.5s_infinite]"></div>
               <Camera size={48} className="text-slate-600" />
               <button onClick={() => { addAddress("75 Avenue des Champs-Élysées, Paris"); setIsCapturing(false); }} className="mt-10 bg-white text-blue-700 px-10 py-4 rounded-2xl font-black">SIMULER CAPTURE</button>
             </div>
          </div>
        </div>
      )}

      <style dangerouslySetInnerHTML={{ __html: `
        @keyframes scan { 0% { top: 15%; opacity: 0; } 20% { opacity: 1; } 80% { opacity: 1; } 100% { top: 85%; opacity: 0; } }
      `}} />
    </div>
  );
};

export default App;
