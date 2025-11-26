<script>
  import { onMount, onDestroy } from "svelte";
  import { parseBlob } from "music-metadata";
  import {
    Battery,
    Wifi,
    Play,
    Pause,
    ChevronRight,
    Music,
    User,
    Code,
    Mail,
    Briefcase,
    Star,
    SkipBack,
    SkipForward,
  } from "lucide-svelte";

  // --- DATA: Estructura del iPod Classic ---
  const PORTFOLIO_DATA = {
    title: "iPod",
    items: [
      {
        label: "Music",
        type: "music-loader",
        icon: Music,
        children: [],
      },
      {
        label: "Photos",
        type: "unavailable",
        icon: User,
      },
      {
        label: "Videos",
        type: "unavailable",
        icon: Play,
      },
      {
        label: "Extras",
        type: "menu",
        icon: Star,
        children: [
          {
            label: "Clock",
            type: "unavailable",
            icon: Star,
          },
          {
            label: "Games",
            type: "unavailable",
            icon: Code,
          },
          {
            label: "Notes",
            type: "unavailable",
            icon: Mail,
          },
          {
            label: "Contacts",
            type: "unavailable",
            icon: User,
          },
          {
            label: "Calendar",
            type: "unavailable",
            icon: Star,
          },
          {
            label: "Voice Memos",
            type: "unavailable",
            icon: Music,
          },
        ],
      },
      {
        label: "Settings",
        type: "menu",
        icon: Code,
        children: [
          {
            label: "About",
            type: "unavailable",
            icon: Star,
          },
          {
            label: "Main Menu",
            type: "unavailable",
            icon: Music,
          },
          {
            label: "Music",
            type: "unavailable",
            icon: Music,
          },
          {
            label: "Videos",
            type: "unavailable",
            icon: Play,
          },
          {
            label: "Photos",
            type: "unavailable",
            icon: User,
          },
          {
            label: "Podcasts",
            type: "unavailable",
            icon: Music,
          },
          {
            label: "Audiobooks",
            type: "unavailable",
            icon: Music,
          },
          {
            label: "Date & Time",
            type: "unavailable",
            icon: Star,
          },
          {
            label: "Language",
            type: "unavailable",
            icon: Star,
          },
          {
            label: "Legal",
            type: "unavailable",
            icon: Mail,
          },
          {
            label: "Reset Settings",
            type: "unavailable",
            icon: Code,
          },
        ],
      },
      {
        label: "Shuffle Songs",
        type: "unavailable",
        icon: Music,
      },
      {
        label: "Now Playing",
        type: "unavailable",
        icon: Play,
      },
    ],
  };

  // --- STATE ---
  let menuStack = [
    { title: PORTFOLIO_DATA.title, items: PORTFOLIO_DATA.items },
  ];
  let selectedIndex = 0;
  let activeSong = null;
  let isPlaying = false;
  let currentTime = "";
  let fileInputRef;
  let folderInputRef;
  let loadedSongs = [];
  let currentSongIndex = 0;
  let isLoadingSongInfo = false;

  // Audio player state
  let audioElement = undefined;
  let currentPlaybackTime = 0;
  let songDuration = 0;

  // Format time in mm:ss
  function formatTime(seconds) {
    if (!seconds || isNaN(seconds)) return "0:00";
    const mins = Math.floor(seconds / 60);
    const secs = Math.floor(seconds % 60);
    return `${mins}:${secs.toString().padStart(2, "0")}`;
  }

  // Audio event handlers
  function handleTimeUpdate() {
    if (audioElement) {
      currentPlaybackTime = audioElement.currentTime;
      songDuration = audioElement.duration || 0;
    }
  }

  function handleAudioEnded() {
    isPlaying = false;
    currentPlaybackTime = 0;

    // Reproducir siguiente canción si hay más en la lista
    if (loadedSongs.length > 1) {
      playNextSong();
    }
  }

  // Haptic feedback
  function triggerHaptic(intensity = "light") {
    // Vibración estándar para navegadores compatibles
    if (navigator.vibrate) {
      const pattern = {
        light: 10,
        medium: 20,
        heavy: 30,
      };
      navigator.vibrate(pattern[intensity] || 10);
    }

    // Soporte para iOS usando el truco del checkbox
    const switchInput = /** @type {HTMLInputElement} */ (
      document.getElementById("hapticSwitch")
    );
    const label = /** @type {HTMLLabelElement} */ (
      document.querySelector('label[for="hapticSwitch"]')
    );
    if (switchInput && label) {
      switchInput.checked = !switchInput.checked;
      label.click();
    }
  }

  // Wheel State
  let wheelElement;
  let prevAngle = null;

  $: currentMenu = menuStack[menuStack.length - 1].items;

  // --- LOGIC: Navegación ---

  function scrollDown() {
    if (activeSong) return;
    selectedIndex = (selectedIndex + 1) % currentMenu.length;
    triggerHaptic("light");
  }

  function scrollUp() {
    if (activeSong) return;
    selectedIndex =
      (selectedIndex - 1 + currentMenu.length) % currentMenu.length;
    triggerHaptic("light");
  }

  function handleSelect() {
    if (activeSong) return;
    triggerHaptic("medium");

    const item = currentMenu[selectedIndex];

    if (item.type === "menu") {
      menuStack = [...menuStack, { title: item.label, items: item.children }];
      selectedIndex = 0;
    } else if (item.type === "music-loader") {
      // Abrir diálogo de selección de archivo
      triggerFileInput();
    } else if (item.type === "unavailable") {
      activeSong = {
        label: item.label,
        artist: "No disponible",
        album: "",
        content: "Esta función no está disponible actualmente.",
        coverColor: "bg-gray-500",
      };
    } else if (item.type === "song") {
      activeSong = item;
      isPlaying = false;
      currentPlaybackTime = 0;

      // Actualizar el índice de la canción actual
      currentSongIndex = loadedSongs.findIndex((song) => song.url === item.url);
      if (currentSongIndex === -1) currentSongIndex = 0;

      // Iniciar reproducción después de que el audioElement esté listo
      setTimeout(() => {
        if (audioElement && activeSong?.url) {
          audioElement.src = activeSong.url;
          audioElement.load();
          audioElement
            .play()
            .then(() => {
              isPlaying = true;
            })
            .catch((error) => {
              console.error("Error al reproducir:", error);
            });
        }
      }, 100);
    } else if (item.type === "text") {
      activeSong = { ...item, artist: "Información", album: "Detalle" };
    }
  }

  function handleMenu() {
    triggerHaptic("medium");
    if (activeSong) {
      activeSong = null;
      isPlaying = false;
      if (audioElement) {
        audioElement.pause();
        audioElement.currentTime = 0;
      }
      return;
    }

    if (menuStack.length > 1) {
      const newStack = [...menuStack];
      newStack.pop();
      menuStack = newStack;
      selectedIndex = 0;
    }
  }

  function handlePlayPause() {
    if (!audioElement || !activeSong?.url) return;
    triggerHaptic("medium");

    if (isPlaying) {
      audioElement.pause();
      isPlaying = false;
    } else {
      audioElement
        .play()
        .then(() => {
          isPlaying = true;
        })
        .catch((error) => {
          console.error("Error al reproducir:", error);
        });
    }
  }

  // --- LOGIC: Manejo de archivos MP3 ---

  async function processSingleFile(file) {
    // Extraer título y artista del nombre del archivo
    const fileName = file.name.replace(".mp3", "");
    let title = fileName;
    let artist = "Artista Desconocido";

    // Intentar parsear el formato "Artista - Título"
    if (fileName.includes(" - ")) {
      const parts = fileName.split(" - ");
      artist = parts[0].trim();
      title = parts.slice(1).join(" - ").trim();
    }

    // Intentar obtener la carátula del archivo MP3
    let coverArt = null;
    try {
      const metadata = await parseBlob(file);
      if (metadata.common.picture && metadata.common.picture.length > 0) {
        const picture = metadata.common.picture[0];
        const blob = new Blob([new Uint8Array(picture.data)], {
          type: picture.format,
        });
        coverArt = URL.createObjectURL(blob);
      }
    } catch (metadataError) {
      console.log("No se pudieron leer los metadatos:", metadataError);
    }

    // Crear objeto de canción
    return {
      label: title,
      artist: artist,
      coverArt: coverArt,
      coverColor: "bg-blue-500",
      type: "song",
      icon: Music,
      file: file,
      url: URL.createObjectURL(file),
    };
  }

  async function handleFileSelect(event) {
    const files = Array.from(event.target.files);
    if (!files || files.length === 0) {
      alert("Por favor selecciona al menos un archivo MP3");
      return;
    }

    // Filtrar solo archivos MP3
    const mp3Files = files.filter((file) => file.name.endsWith(".mp3"));

    if (mp3Files.length === 0) {
      alert("No se encontraron archivos MP3");
      return;
    }

    isLoadingSongInfo = true;

    try {
      // Procesar todos los archivos MP3
      const songPromises = mp3Files.map((file) => processSingleFile(file));
      const songs = await Promise.all(songPromises);

      // Ordenar canciones alfabéticamente por título
      songs.sort((a, b) => a.label.localeCompare(b.label));

      // Actualizar el menú de Music con las canciones cargadas
      loadedSongs = songs;
      currentSongIndex = 0;

      // Navegar al menú de canciones
      const musicMenuItem = PORTFOLIO_DATA.items.find(
        (item) => item.label === "Music",
      );
      if (musicMenuItem) {
        musicMenuItem.children = songs;
        menuStack = [
          ...menuStack,
          { title: `Songs (${songs.length})`, items: songs },
        ];
        selectedIndex = 0;
      }

      isLoadingSongInfo = false;
    } catch (error) {
      console.error("Error al procesar los archivos:", error);
      isLoadingSongInfo = false;
      alert("Error al cargar los archivos MP3");
    }
  }

  function triggerFileInput() {
    // Mostrar opciones de selección
    const choice = confirm(
      "¿Quieres seleccionar una CARPETA completa?\n\nOK = Carpeta\nCancelar = Archivos individuales",
    );

    if (choice) {
      triggerFolderInput();
    } else {
      triggerFilesInput();
    }
  }

  function triggerFilesInput() {
    if (fileInputRef) {
      // Resetear para permitir seleccionar los mismos archivos de nuevo
      fileInputRef.value = "";
      fileInputRef.click();
    }
  }

  function triggerFolderInput() {
    if (folderInputRef) {
      // Resetear para permitir seleccionar la misma carpeta de nuevo
      folderInputRef.value = "";
      folderInputRef.click();
    }
  }

  // Navegación entre canciones
  function playNextSong() {
    if (!activeSong || loadedSongs.length <= 1) return;

    currentSongIndex = (currentSongIndex + 1) % loadedSongs.length;
    playSongAtIndex(currentSongIndex);
  }

  function playPreviousSong() {
    if (!activeSong || loadedSongs.length <= 1) return;

    currentSongIndex =
      (currentSongIndex - 1 + loadedSongs.length) % loadedSongs.length;
    playSongAtIndex(currentSongIndex);
  }

  function playSongAtIndex(index) {
    if (index < 0 || index >= loadedSongs.length) return;

    triggerHaptic("medium");
    activeSong = loadedSongs[index];
    isPlaying = false;
    currentPlaybackTime = 0;

    setTimeout(() => {
      if (audioElement && activeSong?.url) {
        audioElement.src = activeSong.url;
        audioElement.load();
        audioElement
          .play()
          .then(() => {
            isPlaying = true;
          })
          .catch((error) => {
            console.error("Error al reproducir:", error);
          });
      }
    }, 100);
  }

  // --- LOGIC: Click Wheel (Magia Matemática) ---
  function handleWheelMove(e) {
    if (!wheelElement) return;

    const clientX = e.touches ? e.touches[0].clientX : e.clientX;
    const clientY = e.touches ? e.touches[0].clientY : e.clientY;

    const rect = wheelElement.getBoundingClientRect();
    const centerX = rect.left + rect.width / 2;
    const centerY = rect.top + rect.height / 2;

    const x = clientX - centerX;
    const y = clientY - centerY;
    let angle = Math.atan2(y, x) * (180 / Math.PI);
    if (angle < 0) angle += 360;

    if (prevAngle !== null) {
      const delta = angle - prevAngle;

      if (Math.abs(delta) > 10 && Math.abs(delta) < 300) {
        // Ignorar saltos grandes
      } else {
        if (delta > 2 || delta < -300) {
          scrollDown();
          prevAngle = angle;
        } else if (delta < -2 || delta > 300) {
          scrollUp();
          prevAngle = angle;
        }
      }
    } else {
      prevAngle = angle;
    }
  }

  function handleWheelEnd() {
    prevAngle = null;
  }

  function handleWheelStart(e) {
    prevAngle = 0;
    handleWheelMove(e);
  }

  // Event listeners
  let timeInterval;

  onMount(() => {
    timeInterval = setInterval(() => {
      const now = new Date();
      currentTime = now.toLocaleTimeString([], {
        hour: "2-digit",
        minute: "2-digit",
      });
    }, 1000);

    // Mouse events
    const onMouseMove = (e) => {
      if (prevAngle !== null) handleWheelMove(e);
    };
    const onMouseUp = () => handleWheelEnd();

    // Touch events
    const onTouchMove = (e) => {
      if (prevAngle !== null) handleWheelMove(e);
    };

    window.addEventListener("mousemove", onMouseMove);
    window.addEventListener("mouseup", onMouseUp);
    window.addEventListener("touchmove", onTouchMove, { passive: false });
    window.addEventListener("touchend", handleWheelEnd);

    return () => {
      window.removeEventListener("mousemove", onMouseMove);
      window.removeEventListener("mouseup", onMouseUp);
      window.removeEventListener("touchmove", onTouchMove);
      window.removeEventListener("touchend", handleWheelEnd);
    };
  });

  onDestroy(() => {
    if (timeInterval) clearInterval(timeInterval);
  });
</script>

<div
  class="min-h-screen bg-gray-200 flex items-center justify-center p-4 font-sans select-none"
>
  <!-- EL DISPOSITIVO (CHASIS) -->
  <div
    class="relative w-[340px] h-[550px] bg-gradient-to-br from-gray-100 to-gray-300 rounded-[35px] shadow-2xl p-6 border border-white flex flex-col gap-6 transform transition-transform hover:scale-[1.01] duration-500"
  >
    <!-- Acabado Metálico / Brillo -->
    <div
      class="absolute inset-0 rounded-[35px] bg-gradient-to-r from-transparent via-white/40 to-transparent pointer-events-none z-50 opacity-20"
    ></div>

    <!-- PANTALLA -->
    <div
      class="w-full h-64 bg-black rounded-lg border-2 border-gray-400 overflow-hidden shadow-inner relative flex flex-col"
    >
      <!-- Backlight simulado -->
      <div
        class="absolute inset-0 bg-white opacity-[0.05] pointer-events-none z-20 bg-gradient-to-b from-transparent to-black/30"
      ></div>

      <!-- Status Bar -->
      <div
        class="h-6 bg-gradient-to-b from-gray-100 to-gray-200 border-b border-gray-300 flex justify-between items-center px-2 text-[10px] text-gray-800 font-semibold shadow-sm"
      >
        <div class="flex items-center gap-1">
          {#if isPlaying}
            <Play size={10} fill="currentColor" />
          {:else}
            <Pause size={10} fill="currentColor" />
          {/if}
          <span>iPod</span>
        </div>
        <span>{currentTime}</span>
        <div class="flex items-center gap-1">
          <Wifi size={10} />
          <Battery size={12} class="rotate-90" />
        </div>
      </div>

      <!-- Contenido Principal -->
      <div class="flex-1 overflow-hidden relative">
        {#if activeSong}
          <!-- Vista de "Reproducción" (Detalle de Proyecto) -->
          <div class="flex flex-col h-full bg-white">
            <div
              class="flex-1 flex flex-col items-center justify-center p-4 bg-gradient-to-br from-gray-800 to-black text-white relative overflow-hidden"
            >
              <!-- Reflejo simulado -->
              <div
                class="absolute top-0 left-0 w-full h-1/2 bg-white opacity-5 pointer-events-none"
              ></div>

              <div
                class="{activeSong.coverColor ||
                  'bg-gray-500'} w-32 h-32 shadow-2xl mb-4 flex items-center justify-center rounded-sm border border-white/20 overflow-hidden"
              >
                {#if activeSong.coverArt}
                  <img
                    src={activeSong.coverArt}
                    alt={activeSong.label}
                    class="w-full h-full object-cover"
                  />
                {:else}
                  <Music size={48} class="text-white/50" />
                {/if}
              </div>

              <h2 class="text-sm font-bold truncate w-full text-center">
                {activeSong.label}
              </h2>
              <p class="text-xs text-gray-400 truncate w-full text-center">
                {activeSong.artist || ""}
              </p>
              {#if loadedSongs.length > 1}
                <p class="text-[9px] text-gray-500 mb-1">
                  {currentSongIndex + 1} of {loadedSongs.length}
                </p>
              {/if}
              <p class="text-[10px] text-gray-500 mb-2">
                {activeSong.album || ""}
              </p>

              <div
                class="w-full bg-gray-700 h-1.5 rounded-full overflow-hidden mt-2"
              >
                <div
                  class="h-full bg-blue-500 {isPlaying
                    ? 'transition-all duration-1000'
                    : ''}"
                  style="width: {songDuration > 0
                    ? (currentPlaybackTime / songDuration) * 100
                    : 0}%"
                ></div>
              </div>
              <div
                class="flex justify-between w-full text-[9px] text-gray-400 mt-1"
              >
                <span>{formatTime(currentPlaybackTime)}</span>
                <span>-{formatTime(songDuration - currentPlaybackTime)}</span>
              </div>
            </div>

            {#if activeSong.details || activeSong.content}
              <div class="p-3 bg-white text-black min-h-[80px]">
                <h3
                  class="text-[10px] font-bold text-gray-500 uppercase tracking-wider mb-1"
                >
                  Descripción
                </h3>
                <p class="text-xs leading-relaxed">
                  {activeSong.details || activeSong.content}
                </p>
              </div>
            {/if}
          </div>
        {:else}
          <!-- Menú Lista (Split Screen estilo iPod Classic) -->
          <div class="flex h-full bg-white">
            <!-- Lado Izquierdo: Menú -->
            <div class="w-1/2 flex flex-col border-r border-gray-300">
              <div
                class="bg-gradient-to-b from-gray-100 to-gray-200 px-2 py-1 border-b border-gray-300 text-xs font-bold text-center truncate text-gray-700 shadow-sm z-10"
              >
                {menuStack[menuStack.length - 1].title || "Menu"}
              </div>
              <div class="overflow-hidden flex-1">
                {#each currentMenu as item, index}
                  <div
                    class="flex justify-between items-center px-2 py-1.5 text-xs font-medium cursor-pointer transition-colors {index ===
                    selectedIndex
                      ? 'bg-gradient-to-t from-blue-600 to-blue-500 text-white'
                      : 'text-black bg-white'}"
                  >
                    <div class="flex items-center gap-2 truncate">
                      {#if item.icon}
                        <span
                          class={index === selectedIndex
                            ? "opacity-100"
                            : "opacity-50"}
                        >
                          <svelte:component this={item.icon} size={16} />
                        </span>
                      {/if}
                      <span class="truncate">{item.label}</span>
                    </div>
                    {#if item.children || item.type === "song" || item.type === "text" || item.type === "unavailable"}
                      <ChevronRight
                        size={12}
                        class={index === selectedIndex
                          ? "text-white"
                          : "text-gray-400"}
                      />
                    {/if}
                  </div>
                {/each}
              </div>
            </div>

            <!-- Lado Derecho: Preview -->
            <div
              class="w-1/2 bg-white flex items-center justify-center relative overflow-hidden"
            >
              <div
                class="w-full h-full flex flex-col items-center justify-center bg-gray-50 p-4 transition-all duration-300"
              >
                {#if currentMenu[selectedIndex]?.type === "song"}
                  <div
                    class="{currentMenu[selectedIndex].coverColor ||
                      'bg-gray-300'} w-24 h-24 shadow-lg rotate-3 rounded-sm flex items-center justify-center overflow-hidden"
                  >
                    {#if currentMenu[selectedIndex].coverArt}
                      <img
                        src={currentMenu[selectedIndex].coverArt}
                        alt={currentMenu[selectedIndex].label}
                        class="w-full h-full object-cover"
                      />
                    {:else}
                      <Music size={32} class="text-white/50" />
                    {/if}
                  </div>
                {:else if currentMenu[selectedIndex]?.type === "text"}
                  <div class="text-center p-2">
                    <p class="text-xs font-bold text-gray-800">
                      {currentMenu[selectedIndex].label}
                    </p>
                    <p class="text-[10px] text-gray-500 mt-2 line-clamp-4">
                      {currentMenu[selectedIndex].content}
                    </p>
                  </div>
                {:else}
                  <div class="opacity-20 text-gray-500">
                    {#if currentMenu[selectedIndex]?.icon}
                      <svelte:component
                        this={currentMenu[selectedIndex].icon}
                        size={64}
                      />
                    {:else}
                      <Music size={64} />
                    {/if}
                  </div>
                {/if}

                <!-- Overlay de reflejo -->
                <div
                  class="absolute inset-0 bg-gradient-to-tr from-black/5 to-transparent pointer-events-none"
                ></div>
              </div>
            </div>
          </div>
        {/if}
      </div>
    </div>

    <!-- CONTROLES (CLICK WHEEL) -->
    <div class="flex-1 flex items-center justify-center relative">
      <!-- El Anillo Sensible al Tacto -->
      <div
        bind:this={wheelElement}
        on:mousedown={handleWheelStart}
        on:touchstart={handleWheelStart}
        class="w-56 h-56 bg-[#f0f0f0] rounded-full shadow-lg relative flex items-center justify-center cursor-pointer active:cursor-grabbing border border-gray-300"
        style="box-shadow: inset 0 0 10px rgba(0,0,0,0.1), 0 10px 20px rgba(0,0,0,0.15)"
      >
        <!-- MENU (Arriba) -->
        <button
          class="absolute top-3 text-xs font-bold text-gray-500 hover:text-black tracking-wide p-4"
          on:click|stopPropagation={handleMenu}
        >
          MENU
        </button>

        <!-- NEXT (Derecha) -->
        <button
          class="absolute right-3 text-gray-500 hover:text-black p-4"
          on:click|stopPropagation={activeSong && loadedSongs.length > 1
            ? playNextSong
            : scrollDown}
        >
          <SkipForward size={20} fill="currentColor" />
        </button>

        <!-- PREV (Izquierda) -->
        <button
          class="absolute left-3 text-gray-500 hover:text-black p-4"
          on:click|stopPropagation={activeSong && loadedSongs.length > 1
            ? playPreviousSong
            : scrollUp}
        >
          <SkipBack size={20} fill="currentColor" />
        </button>

        <!-- PLAY/PAUSE (Abajo) -->
        <button
          class="absolute bottom-3 text-gray-500 hover:text-black p-4 flex gap-0.5"
          on:click|stopPropagation={handlePlayPause}
        >
          <Play size={16} fill="currentColor" />
          <Pause size={16} fill="currentColor" />
        </button>

        <!-- Botón Central (Select) -->
        <button
          class="w-20 h-20 bg-gradient-to-br from-gray-200 to-gray-300 rounded-full shadow-inner border border-gray-300 flex items-center justify-center active:scale-95 transition-transform z-10"
          on:click|stopPropagation={handleSelect}
          style="box-shadow: inset 0 2px 5px rgba(255,255,255,0.8), inset 0 -2px 5px rgba(0,0,0,0.1), 0 5px 10px rgba(0,0,0,0.2)"
        >
        </button>
      </div>
    </div>
  </div>

  <!-- Footer Info -->
  <div class="fixed bottom-4 text-gray-500 text-xs text-center">
    <p>Frontend Demo • Svelte + Tailwind</p>
    <p class="opacity-50">Arrastra en el anillo para navegar</p>
  </div>

  <!-- Input oculto para cargar archivos MP3 -->
  <input
    bind:this={fileInputRef}
    type="file"
    accept=".mp3,audio/mpeg"
    multiple
    on:change={handleFileSelect}
    class="hidden"
  />

  <!-- Input oculto para cargar carpeta de MP3s -->
  <input
    bind:this={folderInputRef}
    type="file"
    webkitdirectory
    on:change={handleFileSelect}
    class="hidden"
  />

  <!-- Elemento de audio para reproducir MP3 -->
  <audio
    bind:this={audioElement}
    on:timeupdate={handleTimeUpdate}
    on:ended={handleAudioEnded}
    on:loadedmetadata={handleTimeUpdate}
    class="hidden"
  >
    Tu navegador no soporta el elemento de audio.
  </audio>

  <!-- Elementos ocultos para soporte háptico en iOS -->
  <input type="checkbox" id="hapticSwitch" style="display: none;" />
  <label for="hapticSwitch" style="display: none;">Haptic Trigger</label>
</div>
