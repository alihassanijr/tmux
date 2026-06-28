# Patch: keep colour / `#` in app-set pane titles

Base: tmux **3.7** (`81f88f85`). One file: **`input.c`**. Independent of the border-frame
patch (`PATCH-outer-border-frame.md`).

## Why
We wanted colours / `#`-style format sequences to survive in pane titles set by the
running application (OSC 0/2 and APC). `screen_set_title(s, title, untrusted)` takes a
3rd arg: `untrusted != 0` runs `clean_name(title, "#")` which **escapes `#`**; `0` runs
`clean_name(title, "")` and leaves `#` (and thus `#[...]` / `#(...)` format directives)
intact, so they get expanded when the title is rendered (e.g. via `pane-border-format`,
`set-titles-string`).

This flips both app-title call sites from `untrusted=1` → `0`.

## Where (anchor on function names — lines drift)
- `input_exit_osc` → `case 0:`/`case 2:` block: `screen_set_title(sctx->s, p, 1)` → `… , 0)`
- `input_exit_apc`: `screen_set_title(sctx->s, ictx->input_buf, 1)` → `… , 0)`

## ⚠️ Security note (decide before re-applying)
App-set titles are normally marked **untrusted** precisely so `#` is neutralised. With `0`,
a program that prints a title containing `#(command)` can get that command run via tmux
format expansion (and `#[...]` style injection) wherever the title is displayed. Only keep
this if you accept that trade-off / control what writes titles. Gated by `allow-set-title`.

## May be obsolete upstream
This was a local workaround. If a newer tmux gains a proper way to keep colour/format in
titles (or changes the `screen_set_title` signature / default), **check before re-applying**
— you may not need it. If the function signature changed, re-evaluate rather than
mechanically flipping an arg.

## Diff (vs tmux 3.7 `81f88f85`)

```diff
diff --git a/input.c b/input.c
index 4710d08a..fda4048e 100644
--- a/input.c
+++ b/input.c
@@ -2701,7 +2701,7 @@ input_exit_osc(struct input_ctx *ictx)
 	case 2:
 		if (wp != NULL &&
 		    options_get_number(wp->options, "allow-set-title") &&
-		    screen_set_title(sctx->s, p, 1)) {
+		    screen_set_title(sctx->s, p, 0)) {
 			notify_pane("pane-title-changed", wp);
 			server_redraw_window_borders(wp->window);
 			server_status_window(wp->window);
@@ -2779,7 +2779,7 @@ input_exit_apc(struct input_ctx *ictx)
 
 	if (wp != NULL &&
 	    options_get_number(wp->options, "allow-set-title") &&
-	    screen_set_title(sctx->s, ictx->input_buf, 1)) {
+	    screen_set_title(sctx->s, ictx->input_buf, 0)) {
 		notify_pane("pane-title-changed", wp);
 		server_redraw_window_borders(wp->window);
 		server_status_window(wp->window);
```
