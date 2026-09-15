---
name: claude-code-hud
description: Tile every live Claude Code session buffer into a grid inside the current window, giving a heads-up view of all running sessions at once. Use when the user asks to see/show/pull up all Claude Code sessions, all Claude buffers, or wants a "HUD" / dashboard / overview of their sessions.
tools: Bash
---

# Claude Code HUD

Show all live Claude Code session buffers side by side as a grid, laid out
inside the currently selected window. Other windows in the frame are left
untouched.

Session buffers are the ones created by `claude-code.el`: major mode
`claude-code-vterm-mode` (or `claude-code-term-mode` / `claude-code-eat-mode`,
depending on the configured terminal backend). All of them report a `mode-name`
of `"Claude Code Session"`, so matching on `mode-name` covers every backend.

Always drive Emacs through `emacsclient` (see the `emacsclient` skill).

## Build the HUD

```sh
emacsclient --eval '
(let* ((bufs (seq-filter
              (lambda (b)
                (equal (buffer-local-value (quote mode-name) b)
                       "Claude Code Session"))
              (buffer-list)))
       (n (length bufs)))
  (if (zerop n)
      "No Claude Code session buffers found"
    (let* ((cols (if (> n 3) 2 1))
           (rows (ceiling (/ (float n) cols)))
           (col-wins (list (selected-window)))
           (wins nil)
           (remaining n))
      ;; Carve the current window into COLS columns.
      (dotimes (_ (1- cols))
        (setq col-wins
              (append col-wins
                      (list (split-window (car (last col-wins))
                                          nil (quote right))))))
      ;; Carve each column into rows.
      (dolist (cw col-wins)
        (let ((take (min rows remaining))
              (w cw))
          (setq remaining (- remaining take))
          (push w wins)
          (dotimes (_ (1- take))
            (setq w (split-window w nil (quote below)))
            (push w wins))))
      (setq wins (nreverse wins))
      ;; Fill the windows.
      (while (and wins bufs)
        (set-window-buffer (car wins) (car bufs))
        (setq wins (cdr wins) bufs (cdr bufs)))
      (balance-windows)
      (format "Displayed %d Claude Code session buffers in a %dx%d grid"
              n cols rows))))'
```

## Notes

- **Layout**: 1 column for up to 3 sessions, otherwise 2 columns; rows are
  `ceil(n / cols)`. `balance-windows` evens out the sizes afterward.
- **Scope**: only the selected window is subdivided — splits never disturb the
  rest of the frame.
- **Ordering**: buffers come back in `buffer-list` order, i.e. most recently
  used first, so the session the user was just in lands in the top-left cell.
- If the user wants a different shape (single column, one row, a dedicated
  frame), adjust `cols`, or wrap the whole form in
  `(with-selected-frame (make-frame) ...)` for a separate HUD frame.
- To inspect what is available before laying anything out:

  ```sh
  emacsclient --eval '
  (mapcar (lambda (b)
            (cons (buffer-name b)
                  (format "%s" (buffer-local-value (quote mode-name) b))))
          (buffer-list))'
  ```
