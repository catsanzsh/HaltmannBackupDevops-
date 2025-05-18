import tkinter as tk
from tkinter import Frame, Canvas, Scrollbar, Entry, Button, Listbox, END, Toplevel, Text
import contextlib
import io
import random
import re

class O3MiniCopycat:
    def __init__(self):
        self.greetings = [
            "Meow! Welcome to CATGPT 🐾 — what shall we vibe about?",
            "Catseek R1 ready, nyah! Type anything to begin.",
            "Hey, cutie! Ready to claw through code or just meme?"
        ]
        self.fallbacks = [
            "Hmm, that's interesting! Want to go deeper?",
            "Mrow? Try rephrasing or share more details.",
            "Not sure I caught that, meow — one more time?"
        ]
        self.jokes = [
            "Why do programmers like cats? Because they take naps on keyboards.",
            "Cats are natural hackers — just look at the paw-ssword they typed!",
            "What's a cat’s favorite button? FUR-mat disk."
        ]
        self.affirmations = [
            "You're the purr-fect coder! Keep going.",
            "Every bug can be tamed with enough purrs.",
            "Remember: If you fits, you ships!"
        ]
        self.code_examples = [
            "Here's a Python cat:\n```python\ndef cat():\n    print('meow!')\n```",
            "Try this:\n```python\nprint('CATGPT purrs at your service!')\n```",
            "Want a for-loop, nyah?\n```python\nfor i in range(3):\n    print('meow', i)\n```"
        ]

    def _intent(self, prompt):
        tokens = prompt.lower().split()
        if any(w in tokens for w in ("hi", "hello", "hey", "meow")):
            return "greet"
        if any(w in tokens for w in ("joke", "pun", "funny")):
            return "joke"
        if any(w in tokens for w in ("sad", "help", "lonely", "depressed", "upset")):
            return "affirm"
        if any(w in tokens for w in ("code", "python", "script", "function", "def")):
            return "code"
        if "?" in prompt or any(w in tokens for w in ("what", "who", "why", "how", "where")):
            return "question"
        return "fallback"

    def generate(self, prompt: str) -> str:
        intent = self._intent(prompt)
        if intent == "greet":
            return random.choice(self.greetings)
        elif intent == "joke":
            return random.choice(self.jokes)
        elif intent == "affirm":
            return random.choice(self.affirmations)
        elif intent == "code":
            return random.choice(self.code_examples)
        elif intent == "question":
            return "That's a good question! But I'm just a cat-bot. Got tuna?"
        else:
            return random.choice(self.fallbacks)

def extract_code_blocks(text: str):
    return re.findall(r"```(?:python)?\n(.*?)```", text, re.DOTALL) or []

class CATGPT:
    SIDEBAR_BG = "#202123"
    SIDEBAR_WIDTH = 210
    MSG_BG_USER = "#343541"
    MSG_BG_ASSIST = "#40414f"
    CODE_BG = "#23252e"

    def __init__(self, root: tk.Tk):
        self.root = root
        root.title("CATGPT • Local")
        root.iconbitmap("")  # Optional: Add a .ico path here
        root.configure(bg=self.SIDEBAR_BG)
        root.geometry("940x660")
        root.minsize(800, 600)

        # Sidebar
        sidebar = Frame(root, bg=self.SIDEBAR_BG, width=self.SIDEBAR_WIDTH)
        sidebar.pack(side="left", fill="y")
        sidebar.pack_propagate(False)

        new_btn = Button(
            sidebar, text="+  New Chat", anchor="w",
            bg="#444654", fg="#ececf1", activebackground="#55596b",
            bd=0, font=("Segoe UI", 11, "bold"), padx=13, pady=8,
            command=self.new_chat
        )
        new_btn.pack(fill="x", pady=(14, 8), padx=10)

        self.chat_list = Listbox(
            sidebar, bg=self.SIDEBAR_BG, fg="#f0f0f0", highlightthickness=0,
            bd=0, activestyle='none', selectbackground="#55596b", font=("Segoe UI", 10)
        )
        self.chat_list.pack(fill="both", expand=True, padx=10, pady=(0, 12))
        self.chat_list.bind("<<ListboxSelect>>", self.on_chat_select)

        # Main Area
        main = Frame(root, bg=self.MSG_BG_USER)
        main.pack(side="right", fill="both", expand=True)

        self.canvas = Canvas(main, bg=self.MSG_BG_USER, highlightthickness=0)
        self.scrollbar = Scrollbar(main, orient="vertical", command=self.canvas.yview)
        self.canvas.configure(yscrollcommand=self.scrollbar.set)
        self.scrollbar.pack(side="right", fill="y")
        self.canvas.pack(side="left", fill="both", expand=True)

        self.msg_frame = Frame(self.canvas, bg=self.MSG_BG_USER)
        self.canvas.create_window((0, 0), window=self.msg_frame, anchor="nw")
        self.msg_frame.bind("<Configure>", lambda e: self.canvas.configure(scrollregion=self.canvas.bbox("all")))

        # Prompt bar
        prompt_bar = Frame(main, bg=self.MSG_BG_USER)
        prompt_bar.pack(fill="x", side="bottom", pady=(2, 6))

        self.entry = Entry(
            prompt_bar, bd=0, relief="flat", font=("Segoe UI", 13),
            bg="#40414f", fg="#ececf1", insertbackground="#ececf1"
        )
        self.entry.pack(fill="both", expand=True, side="left", padx=(12, 4), ipady=10)
        self.entry.bind("<Return>", lambda e: self.send())

        send_btn = Button(
            prompt_bar, text="➤", font=("Segoe UI", 15, "bold"),
            bg="#19c37d", fg="#ffffff", bd=0, relief="flat", padx=14,
            activebackground="#15b26b", command=self.send
        )
        send_btn.pack(side="right", padx=(4, 12), ipady=6)

        self.engine = O3MiniCopycat()
        self.conversations = [[]]  # list of list[(role,text)]
        self.current_conv_idx = 0
        self.refresh_chat_list()
        self._assistant_msg("Meow! Welcome to CATGPT 🐾 — let's code, chat, or just vibe.")

    def _create_bubble(self, text: str, is_user: bool):
        bg = self.MSG_BG_USER if is_user else self.MSG_BG_ASSIST
        anchor = "e" if is_user else "w"
        max_width = 500
        code_blocks = extract_code_blocks(text)
        if code_blocks:
            before = text.split("```", 1)[0].strip()
            if before:
                self._text_label(before, bg, anchor, max_width)
            for code in code_blocks:
                self._code_block(code, anchor)
        else:
            self._text_label(text, bg, anchor, max_width)
        self.canvas.after_idle(lambda: self.canvas.yview_moveto(1.0))

    def _text_label(self, txt: str, bg: str, anchor: str, wrap: int):
        tk.Label(
            self.msg_frame, text=txt, bg=bg, fg="#ececf1", justify="left",
            font=("Segoe UI", 12), wraplength=wrap, padx=14, pady=10,
        ).pack(anchor=anchor, pady=2, padx=24)

    def _code_block(self, code: str, anchor: str):
        frame = Frame(self.msg_frame, bg=self.CODE_BG, bd=1, relief="solid")
        frame.pack(anchor=anchor, padx=32, pady=4, fill="x")
        text_widget = Text(
            frame, bg=self.CODE_BG, fg="#b5e853", font=("Consolas", 11), wrap="none",
            height=min(12, code.count("\n") + 2), borderwidth=0, highlightthickness=0
        )
        text_widget.insert("1.0", code.rstrip())
        text_widget.config(state="disabled")
        text_widget.pack(side="left", fill="both", expand=True, padx=(6, 2), pady=4)
        Button(
            frame, text="Run", bg="#19c37d", fg="#fff", font=("Segoe UI", 9, "bold"),
            bd=0, relief="flat", padx=8, pady=1, activebackground="#15b26b",
            command=lambda c=code: self._run_code_popup(c)
        ).pack(side="right", padx=8, pady=4)

    def _run_code_popup(self, code: str):
        win = Toplevel(self.root)
        win.title("Sandbox Output")
        win.geometry("520x300")
        Frame(win, bg="#1e1e1e").pack(fill="both", expand=True)
        output_box = Text(win, bg="#181e1b", fg="#e2e8f0", font=("Consolas", 12),
                          wrap="word", height=14, width=68)
        output_box.pack(padx=14, pady=16, fill="both", expand=True)
        f = io.StringIO()
        try:
            with contextlib.redirect_stdout(f), contextlib.redirect_stderr(f):
                exec(code, {"__builtins__": {"print": print, "range": range, "len": len, "int": int, "float": float}}, {})
        except Exception as e:
            f.write(f"\nError: {e}\n")
        output = f.getvalue()
        output_box.insert("1.0", output if output.strip() else "[No output]")
        output_box.config(state="disabled")

    def send(self):
        txt = self.entry.get().strip()
        if not txt:
            return
        self.entry.delete(0, END)
        self._user_msg(txt)
        self.root.after(200, lambda: self._assistant_msg(self.engine.generate(txt)))

    def _user_msg(self, text: str):
        self.conversations[self.current_conv_idx].append(("user", text))
        self._create_bubble(text, is_user=True)

    def _assistant_msg(self, text: str):
        self.conversations[self.current_conv_idx].append(("assistant", text))
        self._create_bubble(text, is_user=False)

    def new_chat(self):
        self.current_conv_idx = len(self.conversations)
        self.conversations.append([])
        self.refresh_chat_list()
        for w in self.msg_frame.winfo_children():
            w.destroy()
        self.engine = O3MiniCopycat()
        self._assistant_msg("Meow! New chat started — how can CATGPT help?")

    def refresh_chat_list(self):
        self.chat_list.delete(0, END)
        for i, conv in enumerate(self.conversations):
            if conv:
                title = conv[0][1][:32] + "…" if len(conv[0][1]) > 32 else conv[0][1]
            else:
                title = f"Chat {i + 1}"
            self.chat_list.insert(END, title)
        self.chat_list.select_set(self.current_conv_idx)

    def on_chat_select(self, event):
        if not self.chat_list.curselection():
            return
        idx = self.chat_list.curselection()[0]
        if idx == self.current_conv_idx:
            return
        self.current_conv_idx = idx
        self._load_conversation()

    def _load_conversation(self):
        for w in self.msg_frame.winfo_children():
            w.destroy()
        conv = self.conversations[self.current_conv_idx]
        for role, txt in conv:
            self._create_bubble(txt, is_user=(role == "user"))
        self.canvas.after_idle(lambda: self.canvas.yview_moveto(1.0))
        self.refresh_chat_list()

if __name__ == "__main__":
    tk.Tk.report_callback_exception = lambda *args: None  # suppress noisy tracebacks
    root = tk.Tk()
    CATGPT(root)
    root.mainloop()
