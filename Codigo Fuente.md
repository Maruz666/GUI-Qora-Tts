[GUI-Q-Tts.py](https://github.com/user-attachments/files/29311634/GUI-Q-Tts.py)
#!/usr/bin/env python3
"""
Qora TTS GUI  ─  Interfaz gráfica para qora-tts-windows-x86_64
Python 3.13.0  ·  PyInstaller-ready  ·  100 % tkinter (stdlib)

Empaqueta con:
    pyinstaller --onefile --windowed --name "Qora TTS GUI" qora_tts_gui.py
"""

from __future__ import annotations

import json
import os
import subprocess
import sys
import threading
import tkinter as tk
from pathlib import Path
from tkinter import filedialog, messagebox, ttk

# ──────────────────────────────────────────────────────────────────────────────
#  Configuración persistente (junto al .exe cuando está empaquetado)
# ──────────────────────────────────────────────────────────────────────────────

def _base_dir() -> Path:
    """Devuelve el directorio base del .exe (frozen) o del script."""
    if getattr(sys, "frozen", False):
        return Path(sys.executable).parent
    return Path(__file__).parent


CONFIG_PATH = _base_dir() / "qora_gui_config.json"


def _load_cfg() -> dict:
    if CONFIG_PATH.exists():
        try:
            return json.loads(CONFIG_PATH.read_text(encoding="utf-8"))
        except Exception:
            pass
    return {}


def _save_cfg(cfg: dict) -> None:
    try:
        CONFIG_PATH.write_text(
            json.dumps(cfg, indent=2, ensure_ascii=False), encoding="utf-8"
        )
    except Exception as e:
        messagebox.showerror("Error", f"No se pudo guardar la configuración:\n{e}")


# ──────────────────────────────────────────────────────────────────────────────
#  Datos de los combos
# ──────────────────────────────────────────────────────────────────────────────

SPEAKERS: list[str] = [
    "ryan", "jenny", "guy", "aria", "davis", "jane", "jason", "tony",
    "nancy", "sara", "liam", "emma", "olivia", "noah", "mia", "sophia",
    "isabella", "james", "benjamin", "lucas", "mason", "ethan",
    "alexander", "elijah", "william", "michael", "daniel", "henry",
    "joseph", "samuel",
]

LANGUAGES: list[str] = [
    "spanish", "english", "french", "german", "italian", "portuguese",
    "russian", "japanese", "korean", "chinese", "arabic", "dutch",
    "polish", "swedish", "norwegian", "danish", "finnish", "czech",
    "hungarian", "romanian", "turkish", "greek", "hindi", "catalan",
    "ukrainian",
]

# ──────────────────────────────────────────────────────────────────────────────
#  Colores del tema Catppuccin Mocha (dark)
# ──────────────────────────────────────────────────────────────────────────────

C = {
    "base":    "#1e1e2e",
    "mantle":  "#181825",
    "crust":   "#11111b",
    "surface0": "#313244",
    "surface1": "#45475a",
    "surface2": "#585b70",
    "overlay0": "#6c7086",
    "text":    "#cdd6f4",
    "subtext": "#a6adc8",
    "blue":    "#89b4fa",
    "sky":     "#87ceeb",
    "green":   "#a6e3a1",
    "yellow":  "#f9e2af",
    "red":     "#f38ba8",
    "pink":    "#f5c2e7",
    "mauve":   "#cba6f7",
    "teal":    "#94e2d5",
}


# ──────────────────────────────────────────────────────────────────────────────
#  Aplicación principal
# ──────────────────────────────────────────────────────────────────────────────

class App(tk.Tk):
    def __init__(self) -> None:
        super().__init__()

        self.cfg = _load_cfg()
        self._proc: subprocess.Popen | None = None
        self._running = False

        self._apply_theme()
        self._build_ui()
        self._restore_state()

        # Si no hay ruta guardada → pedir al iniciar
        if not self.cfg.get("qora_dir"):
            self.after(150, self._first_run_dialog)

    # ─── Tema ────────────────────────────────────────────────────────────────

    def _apply_theme(self) -> None:
        self.title("Qora TTS GUI")
        self.minsize(720, 660)
        self.configure(bg=C["base"])

        st = ttk.Style(self)
        st.theme_use("clam")

        # Globales
        st.configure(".",
            background=C["base"],
            foreground=C["text"],
            font=("Segoe UI", 10),
            troughcolor=C["base"],
            borderwidth=0,
            relief="flat",
        )

        # Frames y labels
        for cls in ("TFrame", "TLabel"):
            st.configure(cls, background=C["base"], foreground=C["text"])

        # LabelFrame
        st.configure("TLabelframe",
            background=C["base"],
            bordercolor=C["surface2"],
            relief="solid",
            borderwidth=1,
        )
        st.configure("TLabelframe.Label",
            background=C["base"],
            foreground=C["blue"],
            font=("Segoe UI", 9, "bold"),
        )

        # Entry
        st.configure("TEntry",
            fieldbackground=C["surface0"],
            foreground=C["text"],
            insertcolor=C["text"],
            relief="flat",
            borderwidth=1,
        )
        st.map("TEntry", fieldbackground=[("readonly", C["surface0"])])

        # Combobox
        st.configure("TCombobox",
            fieldbackground=C["surface0"],
            foreground=C["text"],
            selectbackground=C["surface2"],
            arrowcolor=C["blue"],
        )
        st.map("TCombobox",
            fieldbackground=[("readonly", C["surface0"])],
            selectbackground=[("readonly", C["surface2"])],
        )

        # Botón normal
        st.configure("TButton",
            background=C["surface0"],
            foreground=C["text"],
            padding=(10, 5),
        )
        st.map("TButton", background=[("active", C["surface1"])])

        # Botón acento (generar)
        st.configure("Accent.TButton",
            background=C["blue"],
            foreground=C["crust"],
            font=("Segoe UI", 10, "bold"),
            padding=(14, 6),
        )
        st.map("Accent.TButton",
            background=[("active", C["sky"]), ("disabled", C["surface1"])],
            foreground=[("disabled", C["overlay0"])],
        )

        # Botón peligro (detener)
        st.configure("Stop.TButton",
            background=C["red"],
            foreground=C["crust"],
            font=("Segoe UI", 10, "bold"),
            padding=(10, 6),
        )
        st.map("Stop.TButton",
            background=[("active", C["pink"]), ("disabled", C["surface1"])],
            foreground=[("disabled", C["overlay0"])],
        )

        # Scrollbar
        st.configure("TScrollbar",
            background=C["surface0"],
            troughcolor=C["crust"],
            arrowcolor=C["text"],
        )

        # Status bar
        st.configure("Status.TLabel",
            background=C["surface0"],
            foreground=C["green"],
            padding=(8, 4),
        )
        st.configure("StatusBusy.TLabel",
            background=C["surface0"],
            foreground=C["yellow"],
            padding=(8, 4),
        )

    # ─── Construcción de la UI ────────────────────────────────────────────────

    def _build_ui(self) -> None:
        root = ttk.Frame(self, padding=12)
        root.pack(fill=tk.BOTH, expand=True)

        self._build_path_section(root)
        self._build_params_section(root)
        self._build_actions_section(root)
        self._build_preview_section(root)
        self._build_console_section(root)
        self._build_statusbar()

    # -- Sección ruta --------------------------------------------------------

    def _build_path_section(self, parent: ttk.Frame) -> None:
        lf = ttk.LabelFrame(parent, text="  📁  Carpeta qora-tts-windows-x86_64", padding=(10, 7))
        lf.pack(fill=tk.X, pady=(0, 10))

        self.qora_dir_var = tk.StringVar()
        ttk.Entry(lf, textvariable=self.qora_dir_var, state="readonly").pack(
            side=tk.LEFT, fill=tk.X, expand=True, padx=(0, 8)
        )
        ttk.Button(lf, text="Cambiar…", command=self._browse_qora_dir).pack(side=tk.RIGHT)

    # -- Sección parámetros --------------------------------------------------

    def _build_params_section(self, parent: ttk.Frame) -> None:
        lf = ttk.LabelFrame(parent, text="  🎛  Parámetros de síntesis", padding=(12, 8))
        lf.pack(fill=tk.X, pady=(0, 10))
        lf.columnconfigure(1, weight=1)
        lf.columnconfigure(3, weight=1)

        PAD_Y = 5

        # Fila 0: speaker + idioma
        ttk.Label(lf, text="Speaker:").grid(row=0, column=0, sticky=tk.W, padx=(0, 8), pady=PAD_Y)
        self.speaker_var = tk.StringVar(value="ryan")
        ttk.Combobox(
            lf, textvariable=self.speaker_var, values=SPEAKERS, width=22
        ).grid(row=0, column=1, sticky=tk.EW, pady=PAD_Y)

        ttk.Label(lf, text="Idioma:").grid(row=0, column=2, sticky=tk.W, padx=(20, 8), pady=PAD_Y)
        self.language_var = tk.StringVar(value="spanish")
        ttk.Combobox(
            lf, textvariable=self.language_var, values=LANGUAGES, width=22
        ).grid(row=0, column=3, sticky=tk.EW, pady=PAD_Y)

        # Fila 1: archivo de salida
        ttk.Label(lf, text="Salida:").grid(row=1, column=0, sticky=tk.W, padx=(0, 8), pady=PAD_Y)
        out_row = ttk.Frame(lf)
        out_row.grid(row=1, column=1, columnspan=3, sticky=tk.EW, pady=PAD_Y)
        out_row.columnconfigure(0, weight=1)
        self.output_var = tk.StringVar(value="salida.wav")
        ttk.Entry(out_row, textvariable=self.output_var).grid(
            row=0, column=0, sticky=tk.EW, padx=(0, 6)
        )
        ttk.Button(out_row, text="📂", command=self._browse_output, width=3).grid(row=0, column=1)
        ttk.Label(out_row,
            text="(ruta relativa a la carpeta de qora-tts o ruta absoluta)",
            foreground=C["overlay0"], font=("Segoe UI", 8),
        ).grid(row=1, column=0, sticky=tk.W, pady=(1, 0))

        # Fila 2: texto
        ttk.Label(lf, text="Texto:").grid(row=2, column=0, sticky=tk.NW, padx=(0, 8), pady=(PAD_Y, 0))
        txt_wrap = ttk.Frame(lf)
        txt_wrap.grid(row=2, column=1, columnspan=3, sticky=tk.EW, pady=(PAD_Y, 0))
        txt_wrap.columnconfigure(0, weight=1)

        self.text_widget = tk.Text(
            txt_wrap, height=5, wrap=tk.WORD,
            bg=C["surface0"], fg=C["text"],
            insertbackground=C["text"],
            selectbackground=C["surface2"],
            font=("Segoe UI", 10), padx=8, pady=6,
            relief="flat",
            highlightthickness=1,
            highlightbackground=C["surface2"],
            highlightcolor=C["blue"],
        )
        self.text_widget.grid(row=0, column=0, sticky=tk.EW)
        self.text_widget.insert("1.0", "Hola mundo, ¿cómo estás hoy?")
        self.text_widget.bind("<KeyRelease>", self._on_text_change)

        self.char_label = ttk.Label(lf, foreground=C["overlay0"], font=("Segoe UI", 8))
        self.char_label.grid(row=3, column=1, columnspan=3, sticky=tk.E, pady=(2, 0))
        self._on_text_change()

    # -- Sección acciones ----------------------------------------------------

    def _build_actions_section(self, parent: ttk.Frame) -> None:
        row = ttk.Frame(parent)
        row.pack(fill=tk.X, pady=(0, 8))

        self.gen_btn = ttk.Button(
            row, text="▶  Generar Audio",
            style="Accent.TButton",
            command=self._generate,
        )
        self.gen_btn.pack(side=tk.LEFT)

        self.stop_btn = ttk.Button(
            row, text="■  Detener",
            style="Stop.TButton",
            command=self._stop,
            state=tk.DISABLED,
        )
        self.stop_btn.pack(side=tk.LEFT, padx=(8, 0))

        ttk.Button(
            row, text="🗑  Limpiar consola",
            command=self._clear_console,
        ).pack(side=tk.RIGHT)

    # -- Vista previa del comando --------------------------------------------

    def _build_preview_section(self, parent: ttk.Frame) -> None:
        lf = ttk.LabelFrame(parent, text="  📋  Vista previa del comando", padding=(10, 5))
        lf.pack(fill=tk.X, pady=(0, 10))

        self.preview_var = tk.StringVar()
        preview_entry = tk.Entry(
            lf,
            textvariable=self.preview_var,
            state="readonly",
            readonlybackground=C["crust"],
            fg=C["mauve"],
            font=("Consolas", 9),
            relief="flat",
        )
        preview_entry.pack(fill=tk.X)

        # Actualizar preview en vivo
        for v in (self.speaker_var, self.language_var, self.output_var, self.qora_dir_var):
            v.trace_add("write", self._update_preview)
        self.text_widget.bind("<KeyRelease>", self._on_all_change, add="+")
        self._update_preview()

    # -- Consola de debug ----------------------------------------------------

    def _build_console_section(self, parent: ttk.Frame) -> None:
        lf = ttk.LabelFrame(parent, text="  🖥  Consola de depuración", padding=(6, 4))
        lf.pack(fill=tk.BOTH, expand=True)

        self.console = tk.Text(
            lf,
            state=tk.DISABLED,
            bg=C["crust"],
            fg=C["green"],
            selectbackground=C["surface1"],
            font=("Consolas", 9),
            padx=8, pady=6,
            wrap=tk.WORD,
            relief="flat",
        )
        sb = ttk.Scrollbar(lf, command=self.console.yview)
        self.console.configure(yscrollcommand=sb.set)
        sb.pack(side=tk.RIGHT, fill=tk.Y)
        self.console.pack(fill=tk.BOTH, expand=True)

        # Tags de color
        self.console.tag_configure("info",  foreground=C["blue"])
        self.console.tag_configure("ok",    foreground=C["green"])
        self.console.tag_configure("warn",  foreground=C["yellow"])
        self.console.tag_configure("error", foreground=C["red"])
        self.console.tag_configure("cmd",   foreground=C["mauve"])
        self.console.tag_configure("dim",   foreground=C["overlay0"])

    # -- Status bar ----------------------------------------------------------

    def _build_statusbar(self) -> None:
        self.status_var = tk.StringVar(value="✔  Listo.")
        self.status_lbl = ttk.Label(
            self,
            textvariable=self.status_var,
            style="Status.TLabel",
        )
        self.status_lbl.pack(fill=tk.X, side=tk.BOTTOM)

    # ─── Helpers de UI ───────────────────────────────────────────────────────

    def _restore_state(self) -> None:
        if v := self.cfg.get("qora_dir"):
            self.qora_dir_var.set(v)
        if v := self.cfg.get("speaker"):
            self.speaker_var.set(v)
        if v := self.cfg.get("language"):
            self.language_var.set(v)
        if v := self.cfg.get("output"):
            self.output_var.set(v)

    def _first_run_dialog(self) -> None:
        messagebox.showinfo(
            "Configuración inicial",
            "Bienvenido a Qora TTS GUI.\n\n"
            "Selecciona la carpeta 'qora-tts-windows-x86_64'\n"
            "que contiene el archivo qora-tts.exe.",
        )
        self._browse_qora_dir()

    def _browse_qora_dir(self) -> None:
        initial = self.qora_dir_var.get() or os.getcwd()
        d = filedialog.askdirectory(
            title="Seleccionar carpeta qora-tts-windows-x86_64",
            initialdir=initial,
        )
        if d:
            self.qora_dir_var.set(d)
            self.cfg["qora_dir"] = d
            _save_cfg(self.cfg)
            self._log(f"Carpeta configurada: {d}\n", "info")
            self._update_preview()

    def _browse_output(self) -> None:
        f = filedialog.asksaveasfilename(
            title="Guardar audio como…",
            defaultextension=".wav",
            filetypes=[("WAV audio", "*.wav"), ("Todos los archivos", "*.*")],
            initialfile=Path(self.output_var.get()).name,
        )
        if f:
            self.output_var.set(f)

    def _on_text_change(self, event=None) -> None:
        n = len(self.text_widget.get("1.0", tk.END).strip())
        self.char_label.configure(text=f"{n:,} caracteres")

    def _on_all_change(self, event=None) -> None:
        self._on_text_change()
        self._update_preview()

    def _update_preview(self, *_) -> None:
        qdir = self.qora_dir_var.get()
        exe  = str(Path(qdir) / "qora-tts.exe") if qdir else "qora-tts.exe"
        text = self.text_widget.get("1.0", tk.END).strip() if hasattr(self, "text_widget") else ""
        short_text = (text[:40] + "…") if len(text) > 40 else text
        cmd = (
            f'{exe} '
            f'--speaker {self.speaker_var.get()} '
            f'--language {self.language_var.get()} '
            f'--text "{short_text}" '
            f'--output {self.output_var.get()}'
        )
        self.preview_var.set(cmd)

    # ─── Console helpers ─────────────────────────────────────────────────────

    def _log(self, text: str, tag: str = "") -> None:
        self.console.configure(state=tk.NORMAL)
        self.console.insert(tk.END, text, tag)
        self.console.see(tk.END)
        self.console.configure(state=tk.DISABLED)

    def _clear_console(self) -> None:
        self.console.configure(state=tk.NORMAL)
        self.console.delete("1.0", tk.END)
        self.console.configure(state=tk.DISABLED)

    def _set_running(self, running: bool) -> None:
        self._running = running
        self.gen_btn.configure(state=tk.DISABLED if running else tk.NORMAL)
        self.stop_btn.configure(state=tk.NORMAL if running else tk.DISABLED)
        if running:
            self.status_var.set("⏳  Generando audio…")
            self.status_lbl.configure(style="StatusBusy.TLabel")
        else:
            self.status_lbl.configure(style="Status.TLabel")

    # ─── Generación (hilo separado) ───────────────────────────────────────────

    def _validate_inputs(self) -> tuple[str, list[str]] | None:
        """Valida y construye el comando. Retorna (cwd, cmd) o None si hay error."""
        qora_dir = self.qora_dir_var.get().strip()
        if not qora_dir:
            messagebox.showwarning("Sin carpeta",
                "Selecciona la carpeta de qora-tts primero.")
            return None

        exe = Path(qora_dir) / "qora-tts.exe"
        if not exe.exists():
            messagebox.showerror("Ejecutable no encontrado",
                f"No se encontró qora-tts.exe en:\n{qora_dir}\n\n"
                "Verifica que la ruta sea correcta.")
            return None

        text = self.text_widget.get("1.0", tk.END).strip()
        if not text:
            messagebox.showwarning("Texto vacío",
                "Escribe el texto que deseas sintetizar.")
            return None

        speaker  = self.speaker_var.get().strip()
        language = self.language_var.get().strip()
        output   = self.output_var.get().strip() or "salida.wav"

        if not speaker:
            messagebox.showwarning("Speaker vacío", "Indica el speaker.")
            return None
        if not language:
            messagebox.showwarning("Idioma vacío", "Indica el idioma.")
            return None

        cmd = [
            str(exe),
            "--speaker",  speaker,
            "--language", language,
            "--text",     text,
            "--output",   output,
        ]
        return qora_dir, cmd

    def _generate(self) -> None:
        result = self._validate_inputs()
        if result is None:
            return
        cwd, cmd = result

        # Guardar preferencias
        self.cfg.update({
            "speaker":  self.speaker_var.get().strip(),
            "language": self.language_var.get().strip(),
            "output":   self.output_var.get().strip(),
        })
        _save_cfg(self.cfg)

        # Log del comando
        self._log(f"\n{'─' * 64}\n", "dim")
        self._log("▶ Comando:\n", "info")
        self._log(f"  {' '.join(repr(a) for a in cmd)}\n\n", "cmd")

        self._set_running(True)
        threading.Thread(
            target=self._run_subprocess,
            args=(cmd, cwd),
            daemon=True,
        ).start()

    def _run_subprocess(self, cmd: list[str], cwd: str) -> None:
        """Ejecuta el subproceso y transmite su salida a la consola (hilo)."""
        # Flag de Windows para no mostrar ventana de CMD
        kwargs: dict = {}
        if sys.platform == "win32":
            kwargs["creationflags"] = subprocess.CREATE_NO_WINDOW

        try:
            self._proc = subprocess.Popen(
                cmd,
                stdout=subprocess.PIPE,
                stderr=subprocess.STDOUT,   # unir stderr → stdout
                cwd=cwd,
                text=True,
                encoding="utf-8",
                errors="replace",
                bufsize=1,                  # line-buffered
                **kwargs,
            )

            assert self._proc.stdout is not None
            for raw_line in self._proc.stdout:
                line = raw_line.rstrip("\n") + "\n"
                self.after(0, self._route_line, line)

            self._proc.wait()
            rc = self._proc.returncode

        except FileNotFoundError:
            self.after(0, self._log,
                "\n[ERROR] No se encontró qora-tts.exe en la ruta indicada.\n", "error")
            rc = -1
        except Exception as exc:
            self.after(0, self._log, f"\n[EXCEPCIÓN] {exc}\n", "error")
            rc = -1
        finally:
            self._proc = None
            self.after(0, self._on_done, rc)

    def _route_line(self, line: str) -> None:
        """Clasifica cada línea de salida y la colorea en la consola."""
        lower = line.lower()
        if any(k in lower for k in ("error", "fail", "exception", "traceback", "crítico")):
            tag = "error"
        elif any(k in lower for k in ("warn", "advertencia", "warning")):
            tag = "warn"
        elif any(k in lower for k in ("ok", "done", "success", "listo", "saved",
                                       "guardado", "written", "complete", "finished")):
            tag = "ok"
        else:
            tag = ""
        self._log(line, tag)

    def _on_done(self, rc: int) -> None:
        """Callback al terminar el subproceso (hilo principal)."""
        self._set_running(False)
        sep = f"{'─' * 64}\n"
        if rc == 0:
            out = self.output_var.get()
            self._log(f"\n{sep}✅ Finalizado correctamente (código {rc})\n", "ok")
            self.status_var.set(f"✅  Audio guardado → {out}")
        elif rc == -1:
            self._log(f"\n{sep}❌ Error interno al ejecutar el proceso.\n", "error")
            self.status_var.set("❌  Error — revisa la consola.")
        else:
            self._log(f"\n{sep}⚠️  Proceso terminó con código {rc}\n", "warn")
            self.status_var.set(f"⚠️  Código de salida: {rc}")

    # ─── Detener proceso ──────────────────────────────────────────────────────

    def _stop(self) -> None:
        if self._proc is not None:
            try:
                self._proc.terminate()
                self._log("\n⛔  Proceso detenido por el usuario.\n", "warn")
            except Exception as exc:
                self._log(f"\n[ERROR al detener] {exc}\n", "error")
        self._set_running(False)
        self.status_var.set("⛔  Detenido por el usuario.")

    # ─── Cierre seguro ───────────────────────────────────────────────────────

    def on_close(self) -> None:
        if self._running:
            if not messagebox.askyesno(
                "Salir",
                "Hay un proceso en ejecución.\n¿Salir igualmente?",
            ):
                return
            self._stop()
        self.destroy()


# ──────────────────────────────────────────────────────────────────────────────
#  Punto de entrada
# ──────────────────────────────────────────────────────────────────────────────

def main() -> None:
    app = App()
    app.protocol("WM_DELETE_WINDOW", app.on_close)
    app.mainloop()


if __name__ == "__main__":
    main()
