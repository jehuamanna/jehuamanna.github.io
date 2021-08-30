---
layout: post
title: "Emacs: Open lines above and below the cursor"
tags:
 -
---

If you want to create a new line above the current and move the cursor, these are the steps:
- Move to the cursor to begining of the line. `C-a`
- Create a new line. `RET`
- Move the cursor to the previous line. `C-p`
- In order to match the indentation, press `TAB`.

Obviously, this can be done in multiple ways. In a different order, you can write a emacs-lisp function and bind it to `C-S-O`.
```emacs-lisp
(global-set-key (kbd "C-S-O") (lambda ()
				(interactive)
				(previous-line)
				(end-of-line)
				(newline-and-indent)))
```

Similarly to open line below the cursor and indent:
```emacs-lisp
(global-set-key (kbd "C-S-M") (lambda ()
				(interactive)
				(end-of-line)
				(newline-and-indent)))
```

