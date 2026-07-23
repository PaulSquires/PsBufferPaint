# CBufferPaint

The flicker-free drawing surface every control in this family paints through, for
FreeBASIC / Win32 on the AfxNova framework. Originally vendored separately into each
control repo; this is its canonical home, following the
[CVScrollBar](https://github.com/PaulSquires/CVScrollBar) precedent of splitting a
component out once it has more than one consumer. This one has twelve.

Consumers: [CListBox](https://github.com/PaulSquires/CListBox),
[CVScrollBar](https://github.com/PaulSquires/CVScrollBar),
[CHScrollBar](https://github.com/PaulSquires/CHScrollBar),
[CStatusBar](https://github.com/PaulSquires/CStatusBar),
[CTabBar](https://github.com/PaulSquires/CTabBar),
[CTextBox](https://github.com/PaulSquires/CTextBox),
[CMenuBar](https://github.com/PaulSquires/CMenuBar),
[CSplitter](https://github.com/PaulSquires/CSplitter),
[CIconPanel](https://github.com/PaulSquires/CIconPanel),
[CSelectBar](https://github.com/PaulSquires/CSelectBar),
[CToggle](https://github.com/PaulSquires/CToggle), and tiko.

> **Formerly `clsDoubleBuffer`.** Same class, renamed, with the dual GDI/GDI+ backend
> collapsed to GDI+ only. See [Renamed from clsDoubleBuffer](#renamed-from-clsdoublebuffer)
> if you are porting a host across.

## What it does

Wraps the `BeginPaint` → offscreen bitmap → `BitBlt` → `EndPaint` cycle, and exposes a
small set of drawing primitives on top of it. A control's `WM_PAINT` is one object and a
handful of calls:

```freebasic
case WM_PAINT
    dim b as CBufferPaint
    b.BeginDoubleBuffer( hwnd )
    b.SetBackColor( BackColor )
    b.PaintClientRect()
    ' ... draw ...
    b.EndDoubleBuffer()
    return 0
```

Controls hand it to their hosts through a `PAINTINFO.b as CBufferPaint ptr` field, so a
host paint callback never sees an HDC. That indirection is what let the rendering change
underneath twelve controls without any of them being modified.

## Rendering

**Geometry is drawn with GDI+. Text is drawn with GDI.**

| | |
|---|---|
| `PaintRect`, `PaintRoundRect`, `PaintBorderRect`, `PaintRoundBorderRect`, `PaintRoundOutline`, `PaintEllipse`, `PaintLine` | GDI+, antialiased where it helps |
| `PaintText`, `PaintChar` | GDI `DrawText` |

**Why text stays on GDI.** GDI+ measures and lays out text differently, so every
`GetTextExtentPoint32W` / `GetTextMetricsW` site in the consuming controls' `LayoutItems`
would have to convert to `MeasureString` in lockstep or text clips, and icon glyphs
(Segoe Fluent Icons) would shift. That is a separate project.

Sharing one HDC between the two costs exactly one thing, and it is paid inside the class:
**GDI+ batches its drawing and GDI does not**, so any GDI call on the surface must be
preceded by a flush or shapes go missing *intermittently* — the worst failure mode there
is. The flush sits before every text call, inside `getMemDC()`, and before the final blit.
No control or host has to know.

## API

```freebasic
' Lifecycle -- three Begin overloads, one End.
declare function BeginDoubleBuffer( byval hwnd as HWND ) as long
declare function BeginDoubleBuffer( byval hwnd as HWND, byval hdc as HDC, byval rcItem as RECT ) as long
declare function BeginDoubleBuffer( byval hwnd as HWND, byval hdc as HDC, byval rcItem as RECT, byval cachedMemDC as HDC ) as long
declare function EndDoubleBuffer() as long

' State. Painting always uses the CURRENT colors -- hot/hover styling is the caller's
' decision, made BEFORE the paint call.
declare function SetFont( byval hFont as HFONT ) as long          ' caller owns the HFONT
declare function SetForeColor  ( byval clr as COLORREF ) as long
declare function SetBackColor  ( byval clr as COLORREF ) as long
declare function SetPenColor   ( byval clr as COLORREF ) as long
declare function SetColors     ( byval fore as COLORREF, byval back as COLORREF ) as long
declare function SetForeColorA ( byval clr as COLORREF, byval nAlpha as ubyte ) as long
declare function SetBackColorA ( byval clr as COLORREF, byval nAlpha as ubyte ) as long
declare function SetPenColorA  ( byval clr as COLORREF, byval nAlpha as ubyte ) as long

' Primitives.
declare function PaintRect            ( byval rc as RECT ptr ) as long
declare function PaintClientRect      () as long
declare function PaintBorderRect      ( byval rc as RECT ptr, byval nPenWidth as long = 1 ) as long
declare function PaintRoundRect       ( byval rc as RECT ptr, byval nCurvature as long = 20 ) as long
declare function PaintRoundBorderRect ( byval rc as RECT ptr, byval nCurvature as long = 20, byval nPenWidth as long = 1 ) as long
declare function PaintRoundOutline    ( byval rc as RECT ptr, byval nCurvature as long = 20, byval nPenWidth as long = 1 ) as long
declare function PaintEllipse         ( byval rc as RECT ptr, byval nPenWidth as long = 0 ) as long
declare function PaintLine            ( byval nWidth as long, byval nLeft as long, byval nTop as long, byval nRight as long, byval nBottom as long ) as long
declare function PaintText            ( byval wszText as DWSTRING, byval rc as RECT ptr, byval wsStyle as DWORD ) as long
declare function PaintChar            ( byval wszChar as DWSTRING, byval rc as RECT ptr, byval forecolor as COLORREF ) as long
declare function PaintIconButton      ( byval wszText as DWSTRING, byval rc as RECT ptr, byval nCurvature as long = 20 ) as long

' Queries.
declare function rcClient() as RECT
declare function rcClientWidth() as long
declare function rcClientHeight() as long
declare function getMemDC() as HDC
```

### Notes on individual calls

- **`nCurvature` is a DIAMETER, not a radius** — it keeps GDI `RoundRect`'s meaning for
  compatibility, and the halving a GDI+ arc needs happens inside. A pill is a rounded rect
  whose curvature equals its height; both ends then become exact semicircles.
- **`PaintRoundOutline` strokes without filling.** `PaintRoundBorderRect` always paints the
  interior, which erases anything already drawn there — wrong for a focus ring.
- **`PaintLine` draws axis-aligned rules as filled rectangles**, hard-edged, with GDI's
  exclusive endpoint. Only diagonals are antialiased: a smoothed 1px horizontal rule comes
  out grey and blurry.
- **`getMemDC()` flushes the GDI+ batch before returning.** Asking for the raw HDC means
  you are about to use GDI on it.
- **`SetFont` does not take ownership.** The caller creates and destroys the `HFONT`.
- The **cached `BeginDoubleBuffer` overload** draws into a caller-owned memDC and blits
  without deleting it — for per-row painting (CListBox uses it once per visible row).

## Host obligations

1. **Initialize GDI+ before the first paint and shut it down after the last.** Bracket
   your message loop:

   ```freebasic
   dim as ULONG_PTR gdipToken = AfxGdipInit()
   function = frmMain_Show( 0 )
   AfxGdipShutdown( gdipToken )      ' before CoUninitialize: GDI+ leans on COM
   ```

   Every `CGp*` object must be destroyed before shutdown, which is why this brackets the
   loop rather than sitting anywhere earlier.

2. **Do not name anything `ok`.** GDI+'s `Status` enum defines `Ok = 0` in namespace
   `AfxNova`, and every host in this family says `using AfxNova`, so including this file
   puts `Ok` into your namespace and any identifier called `ok` becomes a duplicated
   definition. Five of the eleven sibling demos had a `SelfTest_Check` parameter named
   exactly that and stopped compiling on adoption. This cannot be fixed from inside
   CBufferPaint — it is the host's own `using AfxNova` that exposes the name. The family
   convention is `bOK`.

## Self-test

```
set CBUFFERPAINT_SELFTEST=1 && main.exe
```

26 assertions. It renders through the real public methods into an offscreen surface and
reads the pixels back, so it asserts the *rendering*, not the arithmetic behind it.

**Every expected number was measured, not derived from what the documentation implies** —
see the table below for the cases where the two disagreed. A number here that looks off by
one is far more likely to be the API's real behaviour than a typo.

Build the demo DPI-aware (`build.bat` does; it refuses to build without `main.rc`). A
DPI-unaware process reports 96 DPI whatever the display is doing, and `PaintLine` is the
one primitive that scales its input — so the pen-thickness assertions would pass for the
wrong reason. They derive their expectations through the same `ScaleY` the code uses,
stored in a `LONG` the same way, rather than hardcoding pixel counts.

## Geometry: GDI+ against GDI's behaviour

The class reproduces GDI's pixel extents deliberately, so that twelve controls' hand-tuned
layouts did not have to move. Six differences had to be handled, all measured:

| | |
|---|---|
| Curvature | GDI's is an ellipse **diameter**; a GDI+ arc takes a **radius**. Halve it. |
| Stroke position | A GDI+ pen is centred on its path, so a stroke laid on the fill boundary lands one pixel **left/up** of where GDI put it. Fill and stroke need separate paths, the stroke offset to the pixel centre. |
| `RoundRect` extent | Fills one row/column **short** of its own rect. GDI does this; match it. |
| `Ellipse` extent | Also stops short — but `ExtTextOut`/`ETO_OPAQUE` does **not**. Three primitives that look alike need three different extents. |
| Line endpoints | `LineTo` **excludes** its endpoint; GDI+ `DrawLine` **includes** it. |
| Thin lines | Antialiasing a 1px axis-aligned rule turns it grey. Draw axis-aligned rules as rectangles. |

One GDI behaviour is deliberately **not** reproduced: `LineTo` over-runs its endpoint with
a pen thicker than 1px (a 2px rule from x=10 to x=40 painted 10..40 where the 1px rule
painted 10..39), and produces a ragged edge doing it. That is an end-cap artifact, not
intent; a 2px rule here is an exact rectangle.

## Renamed from clsDoubleBuffer

Two changes, both mechanical for a host:

1. **The type is `CBufferPaint`**, in `CBufferPaint.bi` / `CBufferPaint.inc`. Every method
   name is unchanged, including `BeginDoubleBuffer` / `EndDoubleBuffer` — the object is
   still a double buffer, `CBufferPaint` is just what it is called. The free functions
   `isMouseOverRECT`, `isMouseOverWindow` and `PaintRect( hDC, rc, clr )` are unchanged
   too. The self-test entry point is `CBufferPaint_RunSelfTest`, gated on
   `CBUFFERPAINT_SELFTEST`.

2. **`#define DBUF_GDIPLUS` is gone, along with the GDI renderer behind it.** Nothing a
   host calls changes; the class simply no longer has a second implementation.

That second point had a real cost, and it is recorded here rather than quietly dropped.
The dual backend existed so that an A/B screenshot diff of the migration was a
one-variable experiment, and so that the self-test had an independent oracle — assertions
passing on **both** backends are ground truth, whereas assertions written against one
implementation only snapshot whatever it happens to do. That migration has shipped, so the
reason expired; but the self-test is now a regression net rather than a proof, and the
rollback is a `git revert` rather than a one-line edit.

## Demo

`main.bas` draws a specimen sheet of every primitive so the rendering can be judged by
eye. The rows to look at are the curved ones — rounded rects, the ellipse and the diagonal
rule should have smooth edges, while the square fills and axis-aligned rules stay crisp.

```
build.bat
```
