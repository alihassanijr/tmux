# Patch: full outer border frame + 4-side active highlight

Base: tmux **3.7** (`81f88f85`). All changes in **`screen-redraw.c`** and **`layout.c`**.
Reproduction guide for re-applying on newer tmux. Anchor on **function names**, not
line numbers — they drift. Diffs at the bottom are the authoritative reference.

Separate, independent change for pane-title colours lives in `PATCH-pane-title-colour.md`.

---

## Goal / what this delivers

- Every pane fully boxed on **4 sides**. Window gets a real **outer frame** (`┏┓┗┛`),
  not just borders between panes. Edge panes are boxed against the terminal edge.
- Active (and marked) pane lights up its **whole** surrounding rectangle, on all four
  sides — independent of how the split layout was built (split order no longer matters).
- Works with `pane-border-status top` **and** `bottom`: edge-pane titles ride on the
  outer frame row; the active pane's shared title-border also lights.
- Removed the old **two-pane half-colouring** (`sy/2` split) — was unwanted and fought
  the frame.

## Mental model (why it was broken)

tmux never reserved space around the window: edge panes sat flush at offset `0,0`, so
there was no cell to draw an outer border in. And `screen_redraw_pane_border()` decided
border *ownership* with a pile of position special-cases (window-edge `xoff==0`/`yoff==0`,
`pane-border-status` suppression, two-pane `sy/2` half-split). Those cases meant a pane
only "owned" some of its sides, so:
- the active highlight only lit some sides, and
- with status lines, the side that is a neighbour's title row was never owned → never lit.

Fix is two ideas:
1. **Inset the whole layout by 1 cell** (`layout.c`). The freed edge cells become the
   outer frame, which the existing border-drawing code renders for free once the
   "window edge == implicit border" shortcuts are removed.
2. **Make border ownership + active-highlight geometric** (`screen-redraw.c`), instead
   of the special-case soup. Active highlight is a plain rectangle-ring test; tiled
   border ownership is the same geometric ring as floating panes already used.

Key enabler: **`resize.c`** already does `if (sx < w->layout_root->sx) sx = w->layout_root->sx;`
then `window_resize(w, sx, ...)`. So insetting `layout_root` by 1 keeps `w->sx`/`w->sy`
at full client size — the freed edge cells are the frame. No `resize.c` change needed.

---

## Changes — `layout.c`

1. **Add margin helpers** (top of file, after the header comment). `layout_outer(n)`
   returns the per-side border width (0 if the dimension is too small to fit one),
   `layout_inset(n)` returns the usable size after removing both sides.

2. **`layout_init`**: size the root cell to `layout_inset(w->sx) x layout_inset(w->sy)`
   at offset `layout_outer(w->sx), layout_outer(w->sy)` (was full size at `0,0`).

3. **`layout_resize`**: right after the floating early-return, inset the incoming target:
   `sx = layout_inset(sx); sy = layout_inset(sy);`.

4. **`layout_fix_offsets`**: set the root cell offset to `layout_outer(w->sx/ w->sy)`
   (was hard `0,0`).

5. **`layout_add_horizontal_border`**: return `0` when `layout_outer(w->sy) != 0`
   (i.e. whenever inset). With an outer frame, the topmost/bottommost pane's status
   line lives on the outer frame row, so the old "reserve an extra row" behaviour is
   redundant and otherwise leaves an empty row that renders as fill dots. (Falls back
   to the original behaviour only for windows too small to inset.)

> Not done (follow-ups): **`layout-set.c`** preset layouts (`even-*`, `tiled`, `main-*`)
> still compute from `w->sx/sy` (~31 refs) and are **not** inset → `select-layout` will
> misalign. Manual splits (`split-window`) and new windows are fine.

## Changes — `screen-redraw.c`

6. **`screen_redraw_check_is`** (active/marked highlight): replace the
   `screen_redraw_pane_border()` delegation with a **geometric ring test** on the
   pane's rectangle (`left=xoff-1, right=xoff+sx (+ scrollbar), top=yoff-1, bottom=yoff+sy`).
   Returns 1 if `(px,py)` is on that ring. Lights all four sides regardless of layout/status.
   *(This is also the foundation everything else leans on — apply it first.)*

7. **`screen_redraw_make_pane_status`** (the title bar's colour): use
   `pane-active-border-style` if `wp == active` **or** the status row lies on the active
   pane's border ring (call `screen_redraw_check_is(rctx, wp->xoff+2, statusrow, active)`).
   So the active pane's bottom/top edge — which is a *neighbour's* title bar — lights too.
   `statusrow = wp->yoff-1` for `TOP`, `wp->yoff+wp->sy` for `BOTTOM`.

8. **`screen_redraw_pane_border`** (tiled path): collapse the whole left/right + top/bottom
   special-case block to the **same geometric ring** as the floating-pane branch right
   above it. Drop `hsplit`/`vsplit`/`split_type`/`oo` and the `pane-border-indicators`
   switch. **Keep** the status-line ownership rule so a pane never steals its neighbour's
   title row — *with an exception for the outermost frame row, which carries no title*:
   ```c
   if (py == wp->yoff - 1 &&
       (pane_status != PANE_STATUS_BOTTOM || py == 0))
           return (SCREEN_REDRAW_BORDER_TOP);
   if (py == ey &&
       (pane_status != PANE_STATUS_TOP || py == (int)wp->window->sy - 1))
           return (SCREEN_REDRAW_BORDER_BOTTOM);
   ```
   Keep `ex` (still used by the INSIDE check). Keep `sy` (floating branch uses it).

9. **`screen_redraw_cell_border`**: delete `if (px == sx || py == sy) return (1);`
   — the window edge is no longer an implicit border (the inset frame is a real one).

10. **`screen_redraw_type_of_cell`** (non-floating branch): delete the `px == 0 ||`
    shortcut (left bit) and the `py == 0 ||` shortcut in the `PANE_STATUS_OFF` (`else`)
    branch (top bit). With the frame at real columns/rows `0` and `sx-1`/`sy-1`, the
    glyph/junction is computed correctly from actual neighbour cells. (The `TOP`/`BOTTOM`
    status sub-branches still have `py==0`/`py!=0`/`py!=sy` guards; harmless here, revisit
    if doing deeper status work.)

### Apply order
6 (geometric `check_is`) → 7 (status colour, depends on 6) → 8/9/10 (frame ownership +
drawing) → `layout.c` 1–5. Then build and test.

---

## Test (what "correct" looks like)

```
set -g pane-border-status top        # then repeat with: bottom
set -g pane-border-indicators colour # arrows path is a known follow-up, see below
```
- single pane → full box, all 4 sides including bottom.
- vertical split → box + one divider, no half-lit/half-dotted shared border.
- 2x2 (any split order) → full outer frame + shared inner borders; selecting any pane
  lights its complete rectangle; edge-pane titles sit on the frame; no stray dot row.
- resize terminal → frame tracks all four edges.
- multi-client at different resolutions: the `·` fill on the larger client is normal
  tmux size-mismatch filler, **not** from this patch.

## Known follow-ups (not in this patch)
- **`select-layout` presets** — inset `layout-set.c` (Stage 2).
- **Arrow indicators** (`pane-border-indicators arrows`/`both`) —
  `screen_redraw_draw_border_arrows` has its own two-pane special-casing
  (`screen_redraw_two_panes` + `active == TAILQ_FIRST`) that the new geometry confuses
  (duplicate/misplaced arrows). Untouched here.
- Tiny windows (`< 3` cells in a dimension) degrade gracefully but ugly.

---

## Authoritative diff (vs tmux 3.7 `81f88f85`)

```diff
diff --git a/layout.c b/layout.c
index da98b590..3aeb7fcf 100644
--- a/layout.c
+++ b/layout.c
@@ -33,6 +33,28 @@
  * cell a pointer to its parent cell.
  */
 
+/*
+ * Reserve a one-cell border all the way around the outside of the window so
+ * every edge pane is fully boxed, not just separated from its neighbours. The
+ * layout is inset by this much on each side; the freed edge cells are drawn as
+ * the outer border by screen-redraw.c.
+ */
+#define LAYOUT_OUTER_BORDER 1
+
+/* Outer border width for a given window dimension (0 if too small to fit). */
+static u_int
+layout_outer(u_int n)
+{
+	return (n > 2 * LAYOUT_OUTER_BORDER ? LAYOUT_OUTER_BORDER : 0);
+}
+
+/* Usable layout size for a given window dimension after the outer border. */
+static u_int
+layout_inset(u_int n)
+{
+	return (n - 2 * layout_outer(n));
+}
+
 static u_int	layout_resize_check(struct window *, struct layout_cell *,
 		    enum layout_type);
 static int	layout_resize_pane_grow(struct window *, struct layout_cell *,
@@ -282,8 +304,8 @@ layout_fix_offsets(struct window *w)
 	if (lc->flags & LAYOUT_CELL_FLOATING)
 		return;
 
-	lc->xoff = 0;
-	lc->yoff = 0;
+	lc->xoff = layout_outer(w->sx);
+	lc->yoff = layout_outer(w->sy);
 
 	layout_fix_offsets1(lc);
 }
@@ -340,12 +362,16 @@ layout_cell_is_bottom(struct window *w, struct layout_cell *lc)
 
 /*
  * Returns 1 if we need to add an extra line for the pane status line. This is
- * the case for the most upper or lower panes only.
+ * the case for the most upper or lower panes only - and only when there is no
+ * outer border on that side, since the outer border already provides a row to
+ * host the status line for an edge pane.
  */
 static int
 layout_add_horizontal_border(struct window *w, struct layout_cell *lc,
     int status)
 {
+	if (layout_outer(w->sy) != 0)
+		return (0);
 	if (status == PANE_STATUS_TOP)
 		return (layout_cell_is_top(w, lc));
 	if (status == PANE_STATUS_BOTTOM)
@@ -601,7 +627,8 @@ layout_init(struct window *w, struct window_pane *wp)
 	struct layout_cell	*lc;
 
 	lc = w->layout_root = layout_create_cell(NULL);
-	layout_set_size(lc, w->sx, w->sy, 0, 0);
+	layout_set_size(lc, layout_inset(w->sx), layout_inset(w->sy),
+	    layout_outer(w->sx), layout_outer(w->sy));
 	layout_make_leaf(lc, wp);
 	layout_fix_panes(w, NULL);
 }
@@ -635,6 +662,11 @@ layout_resize(struct window *w, u_int sx, u_int sy)
 	 */
 	if (lc->type == LAYOUT_WINDOWPANE && (lc->flags & LAYOUT_CELL_FLOATING))
 		return;
+
+	/* Inset the layout to leave room for the outer border. */
+	sx = layout_inset(sx);
+	sy = layout_inset(sy);
+
 	xchange = sx - lc->sx;
 	xlimit = layout_resize_check(w, lc, LAYOUT_LEFTRIGHT);
 	if (xchange < 0 && xchange < -xlimit)
diff --git a/screen-redraw.c b/screen-redraw.c
index 17085a41..bf55bea3 100644
--- a/screen-redraw.c
+++ b/screen-redraw.c
@@ -123,12 +123,10 @@ static enum screen_redraw_border_type
 screen_redraw_pane_border(struct screen_redraw_ctx *ctx, struct window_pane *wp,
     int px, int py)
 {
-	struct options	*oo = wp->window->options;
 	int		 ex = wp->xoff + wp->sx, ey = wp->yoff + wp->sy;
-	int		 hsplit = 0, vsplit = 0, pane_status = ctx->pane_status;
+	int		 pane_status = ctx->pane_status;
 	int		 pane_scrollbars = ctx->pane_scrollbars, sb_w = 0;
 	int		 sb_pos, sx = wp->sx, sy = wp->sy, left, right;
-	enum layout_type split_type;
 
 	if (pane_scrollbars != 0)
 		sb_pos = ctx->pane_scrollbars_pos;
@@ -166,76 +164,40 @@ screen_redraw_pane_border(struct screen_redraw_ctx *ctx, struct window_pane *wp,
 		return (SCREEN_REDRAW_OUTSIDE);
 	}
 
-	/* Get pane indicator. */
-	switch (options_get_number(oo, "pane-border-indicators")) {
-	case PANE_BORDER_COLOUR:
-	case PANE_BORDER_BOTH:
-		if (screen_redraw_two_panes(wp->window, &split_type)) {
-			hsplit = (split_type == LAYOUT_LEFTRIGHT);
-			vsplit = (split_type == LAYOUT_TOPBOTTOM);
-		}
-		break;
-	}
-
 	/*
-	 * Left/right borders. The sy / 2 test is to colour only half the
-	 * active window's border when there are two panes.
+	 * Border ring around the pane. Report all four sides (the outer window
+	 * frame and shared inter-pane borders alike) using a plain geometric
+	 * test - the same shape as floating panes above. Active-pane colouring
+	 * is handled by screen_redraw_check_is and the glyph by
+	 * screen_redraw_type_of_cell, so no half-splitting is done here.
+	 *
+	 * The exception is the pane status line: with pane-border-status top
+	 * (bottom) a pane's bottom (top) border is the *next* pane's title
+	 * line, so it must stay owned by that neighbour or its title text would
+	 * be overdrawn. The outermost frame row carries no title, so it is
+	 * always owned.
 	 */
-	if ((wp->yoff == 0 || py >= wp->yoff - 1) && py <= ey) {
-		if (sb_pos == PANE_SCROLLBARS_LEFT) {
-			if (wp->xoff - sb_w == 0 && px == sx + sb_w) {
-				if (!hsplit || (hsplit && py <= sy / 2))
-					return (SCREEN_REDRAW_BORDER_RIGHT);
-			}
-			if (wp->xoff - sb_w != 0) {
-				if (px == wp->xoff - sb_w - 1 &&
-				    (!hsplit || (hsplit && py > sy / 2)))
-					return (SCREEN_REDRAW_BORDER_LEFT);
-				if (px == wp->xoff + sx + sb_w - 1)
-					return (SCREEN_REDRAW_BORDER_RIGHT);
-			}
-		} else { /* sb_pos == PANE_SCROLLBARS_RIGHT or disabled */
-			if (wp->xoff == 0 && px == sx + sb_w) {
-				if (!hsplit || (hsplit && py <= sy / 2))
-					return (SCREEN_REDRAW_BORDER_RIGHT);
-			}
-			if (wp->xoff != 0) {
-				if (px == wp->xoff - 1 &&
-				    (!hsplit || (hsplit && py > sy / 2)))
-					return (SCREEN_REDRAW_BORDER_LEFT);
-				if (px == wp->xoff + sx + sb_w)
-					return (SCREEN_REDRAW_BORDER_RIGHT);
-			}
-		}
-	}
+	left = wp->xoff - 1;
+	right = wp->xoff + sx;
+	if (sb_pos == PANE_SCROLLBARS_LEFT)
+		left -= sb_w;
+	else
+		right += sb_w;
 
-	/* Top/bottom borders. */
-	if (vsplit && pane_status == PANE_STATUS_OFF) {
-		if (wp->yoff == 0 && py == sy && px <= sx / 2)
-			return (SCREEN_REDRAW_BORDER_BOTTOM);
-		if (wp->yoff != 0 && py == wp->yoff - 1 && px > sx / 2)
+	if (py >= wp->yoff - 1 && py <= ey) {
+		if (px == left)
+			return (SCREEN_REDRAW_BORDER_LEFT);
+		if (px == right)
+			return (SCREEN_REDRAW_BORDER_RIGHT);
+	}
+	if (px >= left && px <= right) {
+		if (py == wp->yoff - 1 &&
+		    (pane_status != PANE_STATUS_BOTTOM || py == 0))
 			return (SCREEN_REDRAW_BORDER_TOP);
-	} else {
-		if (sb_pos == PANE_SCROLLBARS_LEFT) {
-			if ((wp->xoff - sb_w == 0 || px >= wp->xoff - sb_w) &&
-			    (px <= ex || (sb_w != 0 && px < ex + sb_w))) {
-				if (pane_status != PANE_STATUS_BOTTOM &&
-				    wp->yoff != 0 && py == wp->yoff - 1)
-					return (SCREEN_REDRAW_BORDER_TOP);
-				if (pane_status != PANE_STATUS_TOP && py == ey)
-					return (SCREEN_REDRAW_BORDER_BOTTOM);
-			}
-		} else { /* sb_pos == PANE_SCROLLBARS_RIGHT */
-			if ((wp->xoff == 0 || px >= wp->xoff) &&
-			    (px <= ex || (sb_w != 0 && px < ex + sb_w))) {
-				if (pane_status != PANE_STATUS_BOTTOM &&
-				    wp->yoff != 0 &&
-				    py == wp->yoff - 1)
-					return (SCREEN_REDRAW_BORDER_TOP);
-				if (pane_status != PANE_STATUS_TOP && py == ey)
-					return (SCREEN_REDRAW_BORDER_BOTTOM);
-			}
-		}
+		if (py == ey &&
+		    (pane_status != PANE_STATUS_TOP ||
+		    py == (int)wp->window->sy - 1))
+			return (SCREEN_REDRAW_BORDER_BOTTOM);
 	}
 
 	/* Outside pane. */
@@ -294,13 +256,16 @@ screen_redraw_cell_border(struct screen_redraw_ctx *ctx, struct window_pane *wp,
 		return (n);
 	}
 
-	/* Outside the window or on the window border? */
+	/*
+	 * Outside the window? The outer border is now a real border drawn
+	 * around the inset layout, so the window edge itself is not treated as
+	 * an implicit border any more - the pane scan below picks up the outer
+	 * border cells like any other.
+	 */
 	if (ctx->pane_status == PANE_STATUS_BOTTOM)
 		sy--;
 	if (px > sx || py > sy)
 		return (0);
-	if (px == sx || py == sy)
-		return (1);
 
 	/*
 	 * If checking a cell from a tiled pane, ignore floating panes because
@@ -344,7 +309,7 @@ screen_redraw_type_of_cell(struct screen_redraw_ctx *ctx,
 	 *		     1
 	 */
 	if (!window_pane_is_floating(wp)) {
-		if (px == 0 || screen_redraw_cell_border(ctx, wp, px - 1, py))
+		if (screen_redraw_cell_border(ctx, wp, px - 1, py))
 			borders |= 8;
 		if (px <= sx && screen_redraw_cell_border(ctx, wp, px + 1, py))
 			borders |= 4;
@@ -362,8 +327,7 @@ screen_redraw_type_of_cell(struct screen_redraw_ctx *ctx,
 			    screen_redraw_cell_border(ctx, wp, px, py + 1))
 				borders |= 1;
 		} else {
-			if (py == 0 ||
-			    screen_redraw_cell_border(ctx, wp, px, py - 1))
+			if (screen_redraw_cell_border(ctx, wp, px, py - 1))
 				borders |= 2;
 			if (screen_redraw_cell_border(ctx, wp, px, py + 1))
 				borders |= 1;
@@ -571,12 +535,36 @@ static int
 screen_redraw_check_is(struct screen_redraw_ctx *ctx, int px, int py,
     struct window_pane *wp)
 {
-	enum screen_redraw_border_type	border;
+	int	left, right, top, bottom, sb_w = 0, sb_pos = 0;
 
 	if (wp == NULL)
 		return (0); /* no active pane */
-	border = screen_redraw_pane_border(ctx, wp, px, py);
-	if (border != SCREEN_REDRAW_INSIDE && border != SCREEN_REDRAW_OUTSIDE)
+
+	/*
+	 * Highlight the whole border rectangle around the active pane,
+	 * regardless of how the layout was built or whether the borders carry
+	 * a pane status line. screen_redraw_pane_border only assigns some
+	 * sides to the pane (it splits a shared border in half, and suppresses
+	 * the status-line edge and the window edges), so use a plain geometric
+	 * test here instead.
+	 */
+	if (ctx->pane_scrollbars != 0)
+		sb_pos = ctx->pane_scrollbars_pos;
+	if (window_pane_show_scrollbar(wp, ctx->pane_scrollbars))
+		sb_w = wp->scrollbar_style.width + wp->scrollbar_style.pad;
+
+	left = wp->xoff - 1;
+	right = wp->xoff + (int)wp->sx;
+	top = wp->yoff - 1;
+	bottom = wp->yoff + (int)wp->sy;
+	if (sb_pos == PANE_SCROLLBARS_LEFT)
+		left -= sb_w;
+	else
+		right += sb_w;
+
+	if ((px == left || px == right) && py >= top && py <= bottom)
+		return (1);
+	if ((py == top || py == bottom) && px >= left && px <= right)
 		return (1);
 	return (0);
 }
@@ -587,6 +575,7 @@ screen_redraw_make_pane_status(struct client *c, struct window_pane *wp,
     struct screen_redraw_ctx *rctx, enum pane_lines pane_lines)
 {
 	struct window		*w = wp->window;
+	struct window_pane	*active = server_client_get_pane(c);
 	struct grid_cell	 gc;
 	const char		*fmt, *border_option;
 	struct format_tree	*ft;
@@ -605,7 +594,21 @@ screen_redraw_make_pane_status(struct client *c, struct window_pane *wp,
 	ft = format_create(c, NULL, FORMAT_PANE|wp->id, FORMAT_STATUS);
 	format_defaults(ft, c, c->session, c->session->curw, wp);
 
-	if (wp == server_client_get_pane(c))
+	/*
+	 * The status line sits on a pane border which is shared with the
+	 * neighbouring pane (its top status line is that neighbour's bottom
+	 * border and vice versa). Use the active border style if this pane is
+	 * active, or if the status line lies on the active pane's border, so
+	 * the active pane is highlighted on all four sides.
+	 */
+	if (pane_status == PANE_STATUS_TOP)
+		py = wp->yoff - 1;
+	else
+		py = wp->yoff + wp->sy;
+	px = wp->xoff + 2;
+	if (wp == active ||
+	    (active != NULL && !window_pane_is_floating(active) &&
+	    screen_redraw_check_is(rctx, px, py, active)))
 		border_option = "pane-active-border-style";
 	else
 		border_option = "pane-border-style";
```
