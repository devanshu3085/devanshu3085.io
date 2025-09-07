# devanshu3085.io
I make website

<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Mini Chess — Drag & Drop</title>
<style>
  :root{
    --light:#f0d9b5; --dark:#b58863;
    --accent:#0b6fb4;
    font-family: Inter, system-ui, sans-serif;
  }
  body{display:flex;align-items:center;justify-content:center;height:100vh;margin:0;background:#0f1724;color:#e6eef8;}
  .container{display:grid;grid-template-columns: 320px 1fr;gap:24px;}
  .board{
    width: 480px; height:480px; display:grid; grid-template-columns: repeat(8,1fr); border-radius:12px; overflow:hidden; box-shadow:0 8px 30px rgba(2,6,23,0.6);
  }
  .square{width:60px;height:60px; display:flex;align-items:center;justify-content:center; font-size:34px; cursor:grab; user-select:none;}
  .square.light{background:var(--light); color:#222}
  .square.dark{background:var(--dark); color:#111}
  .square.highlight{outline:3px solid rgba(11,111,180,0.7); box-sizing:border-box;}
  .square.valid{box-shadow: inset 0 0 0 3px rgba(11,111,180,0.15);}
  .info{width:320px;padding:16px;border-radius:12px;background:linear-gradient(180deg,#071427,#09233a);box-shadow:0 6px 20px rgba(2,6,23,0.6);}
  h2{margin:0 0 12px 0;font-size:18px;color:var(--accent)}
  .meta{font-size:13px; color:#9fb3cc; margin-bottom:12px}
  button{background:var(--accent); color:white;border:0;padding:8px 12px;border-radius:8px;cursor:pointer}
  footer{margin-top:14px;font-size:12px;color:#91b0c8}
  .row{display:flex;gap:8px;flex-wrap:wrap}
</style>
</head>
<body>
<div class="container">
  <div>
    <div id="board" class="board"></div>
  </div>
  <div class="info">
    <h2>Mini Chess</h2>
    <div class="meta">Drag pieces to move. Basic legal moves implemented (no check detection).</div>
    <div><strong>Turn:</strong> <span id="turn">White</span></div>
    <div style="margin-top:8px"><strong>Last Move:</strong> <span id="last">-</span></div>
    <div style="margin-top:12px" class="row">
      <button id="reset">Reset</button>
      <button id="flip">Flip Board</button>
    </div>
    <footer>
      <p>Pieces use Unicode glyphs — extend this to add AI, timers, or FEN export.</p>
    </footer>
  </div>
</div>

<script>
/* Minimal chess logic: board array, moves generation (no check detection) */
const boardEl = document.getElementById('board');
const turnEl = document.getElementById('turn');
const lastEl = document.getElementById('last');
let flipped = false;
let turn = 'w'; // 'w' or 'b'

/* Unicode pieces */
const UNICODE = {
  'r':'♜','n':'♞','b':'♝','q':'♛','k':'♚','p':'♟',
  'R':'♖','N':'♘','B':'♗','Q':'♕','K':'♔','P':'♙'
};

/* Start position (array rows 0..7 top->bottom) */
function startPosition(){
  return [
    ['r','n','b','q','k','b','n','r'],
    ['p','p','p','p','p','p','p','p'],
    ['.','.','.','.','.','.','.','.'],
    ['.','.','.','.','.','.','.','.'],
    ['.','.','.','.','.','.','.','.'],
    ['.','.','.','.','.','.','.','.'],
    ['P','P','P','P','P','P','P','P'],
    ['R','N','B','Q','K','B','N','R']
  ];
}
let board = startPosition();

/* Helpers */
function inBounds(r,c){return r>=0 && r<8 && c>=0 && c<8;}
function isWhitePiece(ch){return ch && ch!=='.' && ch === ch.toUpperCase();}
function isBlackPiece(ch){return ch && ch!=='.' && ch === ch.toLowerCase();}

/* Render */
function renderBoard(){
  boardEl.innerHTML = '';
  for(let r=0;r<8;r++){
    for(let c=0;c<8;c++){
      const sq = document.createElement('div');
      sq.className = 'square ' + ((r+c)%2===0 ? 'light' : 'dark');
      sq.dataset.r = r; sq.dataset.c = c;
      const piece = board[r][c];
      if(piece !== '.') {
        const span = document.createElement('div');
        span.textContent = UNICODE[piece];
        span.draggable = true;
        span.dataset.p = piece;
        span.style.cursor = 'grab';
        span.style.userSelect = 'none';
        span.className = 'piece';
        sq.appendChild(span);
      }
      boardEl.appendChild(sq);
    }
  }
  attachDragHandlers();
}

/* Move generation (basic): returns array of [r,c] destinations */
function genMoves(r,c){
  const piece = board[r][c];
  if(!piece || piece === '.') return [];
  const moves = [];
  const color = isWhitePiece(piece) ? 'w' : 'b';
  const dir = color==='w' ? -1 : 1;

  const addIf = (rr,cc)=>{
    if(!inBounds(rr,cc)) return false;
    const target = board[rr][cc];
    if(target === '.') { moves.push([rr,cc]); return true; }
    if(color === 'w' && isBlackPiece(target)) { moves.push([rr,cc]); return false; }
    if(color === 'b' && isWhitePiece(target)) { moves.push([rr,cc]); return false; }
    return false;
  };

  const p = piece.toLowerCase();
  if(p === 'p'){
    // forward
    if(inBounds(r+dir, c) && board[r+dir][c] === '.') {
      moves.push([r+dir,c]);
      // first double move
      if((color==='w' && r===6) || (color==='b' && r===1)){
        if(board[r+2*dir][c]==='.') moves.push([r+2*dir,c]);
      }
    }
    // captures
    for(const dc of [-1,1]){
      const rr = r+dir, cc = c+dc;
      if(inBounds(rr,cc)){
        const t = board[rr][cc];
        if(t !== '.' && ((color==='w' && isBlackPiece(t)) || (color==='b' && isWhitePiece(t)))){
          moves.push([rr,cc]);
        }
      }
    }
  } else if(p === 'n'){
    const deltas = [[-2,-1],[-2,1],[-1,-2],[-1,2],[1,-2],[1,2],[2,-1],[2,1]];
    deltas.forEach(d=>{ const rr=r+d[0], cc=c+d[1]; if(inBounds(rr,cc) && (board[rr][cc]==='.' || (color==='w'?isBlackPiece(board[rr][cc]):isWhitePiece(board[rr][cc])))) moves.push([rr,cc]); });
  } else if(p === 'b' || p==='r' || p==='q'){
    const steps = [];
    if(p==='b' || p==='q') steps.push([-1,-1],[-1,1],[1,-1],[1,1]);
    if(p==='r' || p==='q') steps.push([-1,0],[1,0],[0,-1],[0,1]);
    steps.forEach(d=>{
      let rr=r+d[0], cc=c+d[1];
      while(inBounds(rr,cc)){
        if(board[rr][cc]==='.') { moves.push([rr,cc]); rr+=d[0]; cc+=d[1]; continue; }
        if(color==='w' && isBlackPiece(board[rr][cc])) { moves.push([rr,cc]); }
        if(color==='b' && isWhitePiece(board[rr][cc])) { moves.push([rr,cc]); }
        break;
      }
    });
  } else if(p === 'k'){
    for(let dr=-1;dr<=1;dr++) for(let dc=-1;dc<=1;dc++){
      if(dr===0 && dc===0) continue;
      const rr=r+dr, cc=c+dc;
      if(inBounds(rr,cc) && (board[rr][cc]==='.' || (color==='w'?isBlackPiece(board[rr][cc]):isWhitePiece(board[rr][cc])))) moves.push([rr,cc]);
    }
  }
  return moves;
}

/* Drag & drop */
let dragSrc = null;
function attachDragHandlers(){
  const pieces = document.querySelectorAll('.piece');
  pieces.forEach(p=>{
    p.addEventListener('dragstart', ev=>{
      const sq = ev.target.parentElement;
      const r = parseInt(sq.dataset.r), c = parseInt(sq.dataset.c);
      // only allow dragging on player's turn
      const piece = board[r][c];
      if((turn === 'w' && !isWhitePiece(piece)) || (turn === 'b' && !isBlackPiece(piece))){
        ev.preventDefault(); return;
      }
      dragSrc = {r,c};
      ev.dataTransfer.setData('text/plain', `${r},${c}`);
      // highlight valid moves
      const moves = genMoves(r,c);
      document.querySelectorAll('.square').forEach(s=>s.classList.remove('valid','highlight'));
      moves.forEach(m=>{
        const sel = document.querySelector(`.square[data-r="${m[0]}"][data-c="${m[1]}"]`);
        if(sel) sel.classList.add('valid');
      });
      sq.classList.add('highlight');
    });
  });

  document.querySelectorAll('.square').forEach(sq=>{
    sq.addEventListener('dragover', ev=>{ ev.preventDefault(); });
    sq.addEventListener('drop', ev=>{
      ev.preventDefault();
      const dstR = parseInt(sq.dataset.r), dstC = parseInt(sq.dataset.c);
      if(!dragSrc) return;
      const sr = dragSrc.r, sc = dragSrc.c;
      const valid = genMoves(sr,sc).some(m=>m[0]===dstR && m[1]===dstC);
      if(valid){
        makeMove(sr,sc,dstR,dstC);
      }
      // cleanup
      document.querySelectorAll('.square').forEach(s=>s.classList.remove('valid','highlight'));
      dragSrc = null;
    });
    sq.addEventListener('dragenter', ev=>{ ev.preventDefault(); });
  });
}

/* Perform move */
function makeMove(sr,sc,dstR,dstC){
  const piece = board[sr][sc];
  const captured = board[dstR][dstC];
  board[dstR][dstC] = piece;
  board[sr][sc] = '.';
  // pawn promotion basic: promote to queen if reaching last rank
  if(piece.toLowerCase() === 'p'){
    if((piece === 'P' && dstR===0) || (piece==='p' && dstR===7)){
      board[dstR][dstC] = piece === 'P' ? 'Q' : 'q';
    }
  }
  // update UI
  lastEl.textContent = `${posToAlgebraic(sr,sc)} → ${posToAlgebraic(dstR,dstC)}${captured!=='.' ? ' x' : ''}`;
  turn = (turn === 'w') ? 'b' : 'w';
  turnEl.textContent = (turn === 'w') ? 'White' : 'Black';
  renderBoard();
}

/* Utilities */
function posToAlgebraic(r,c){
  const files = 'abcdefgh';
  return files[c] + (8 - r);
}

/* Controls */
document.getElementById('reset').addEventListener('click', ()=>{
  board = startPosition(); turn='w'; turnEl.textContent='White'; lastEl.textContent='-'; renderBoard();
});
document.getElementById('flip').addEventListener('click', ()=>{
  flipped = !flipped;
  boardEl.style.transform = flipped ? 'rotate(180deg)' : '';
  // rotate squares' inner content back so pieces are readable
  document.querySelectorAll('.square').forEach(s=>{
    if(flipped) s.style.transform = 'rotate(180deg)'; else s.style.transform = '';
  });
});

/* initial render */
renderBoard();
</script>
</body>
</html>****
