//--------------------------------------DEFINITIONS--------------------------------------//


const debug = true;
const MAX_DELAY = 2; //sec
const MAX_ANALYZE_TIME = 4; //sec

var loaded = {
    "fen": false,
    "stockfish": false,
}

var evaler;

const MAPS = {
    fen: {
        'wr': 'R',
        'wn': 'N',
        'wb': 'B',
        'wq': 'Q',
        'wk': 'K',
        'wp': 'P',
        'br': 'r',
        'bn': 'n',
        'bb': 'b',
        'bq': 'q',
        'bk': 'k',
        'bp': 'p'
    },
    rank:
    {
        'a': '1',
        'b': '2',
        'c': '3',
        'd': '4',
        'e': '5',
        'f': '6',
        'g': '7',
        'h': '8'
    }
}

var peices = {
    "from": undefined,
    "to": undefined
}

//--------------------------------------GETTING CURRENT GAME FEN / GETTING TURN--------------------------------------//

var _fen = ""
for (let i = 8; i >= 1; i--) {
    const element = document.querySelector(`#board-single > square-${i}8`)
    if (element == null) break;
    _fen = _fen + MAPS.fen[element.classList[1]]
}
const __fen = _fen.toUpperCase()
const team = _fen[0] != "R" ? "w" : "b"
const Turn = document.querySelector(".clock-bottom").classList.contains("clock-player-turn") ? team : (team == "w" ? "b" : "w")

//THE OPPOSITE TEAM MIGHT HAVE TAKEN THE KNIGHT OUT FIRST BEFORE THE CODE INITLIZED THE FEN SO, if there is break when the element isn't found as an result we can check if the length is more than 8 if it is we have successfully got the all positions else the rare condition aplied so we have to use the deafult fen expecting the deafult fe is in the starting
const startFen = _fen.length == 8 ? _fen + "/pppppppp/8/8/8/8/PPPPPPPP/" + __fen + " " + Turn + " " + "KQkq - 0 1" : "rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR " + Turn + " " + "KQkq - 0 1"

if (debug) {
    console.log("Team :", team);
    console.log("Turn :", Turn);
    console.log("Refrence FEN :", "length > ", _fen.length, "condition >", _fen.length == 8);
    console.log("FEN :", startFen);
}

//--------------------------------------STYLES [SQARE]--------------------------------------//

const element = document.createElement("style");
element.textContent = `
.SelectedPeice {
    border-style: solid;
    border-width: 4px;
    border-color: blue;
}
  `
document.head.appendChild(element);

//--------------------------------------FUNCTIONS--------------------------------------//


function loadScript(url, callback) {
    let script = document.createElement('script');
    script.src = url;
    script.onload = callback;
    document.head.appendChild(script);
}

function getFen() {
    const moveListContainer = document.querySelector('wc-move-list').children[0];
    // Get all move elements
    const moveElements = moveListContainer.querySelectorAll('.move');

    // Extract move data
    const moveData = [];
    moveElements.forEach(moveElement => {
        const moveNumber = moveElement.getAttribute('data-whole-move-number');
        const whiteMoveElement = moveElement.querySelector('.white.node');
        const blackMoveElement = moveElement.querySelector('.black.node');

        const whiteMove = whiteMoveElement?.innerText.trim() || '';
        const blackMove = blackMoveElement?.innerText.trim() || '';

        let whiteFigurine = '';
        let blackFigurine = '';

        if (whiteMoveElement && whiteMoveElement.querySelector('span.icon-font-chess')) {
            const whiteIcon = whiteMoveElement.querySelector('span.icon-font-chess');
            whiteFigurine = whiteIcon.getAttribute('data-figurine') || '';
        }

        if (blackMoveElement && blackMoveElement.querySelector('span.icon-font-chess')) {
            const blackIcon = blackMoveElement.querySelector('span.icon-font-chess');
            blackFigurine = blackIcon.getAttribute('data-figurine') || '';
        }

        const formattedWhiteMove = whiteFigurine ? `${whiteFigurine}${whiteMove}` : whiteMove;
        const formattedBlackMove = blackFigurine ? `${blackFigurine}${blackMove}` : blackMove;

        const formattedMove = `${moveNumber}. ${formattedWhiteMove} ${formattedBlackMove}`;
        moveData.push(formattedMove);
    });
    // Convert to FEN
    Init(startFen);
    SetPgnMoveText(moveData.join(" "));
    var ff = "", ff_new = "", ff_old;
    do {
        ff_old = ff_new;
        MoveForward(1);
        ff_new = GetFEN();
        if (ff_old != ff_new) ff += ff_new + "\n";
    }
    while (ff_old != ff_new);
    // gives more than one so,
    _ff = ff.split('\n')
    ff = _ff[_ff.length - 2]
    return ff
}
//--------------------------------------PGN TO FEN CONVETER MODULE--------------------------------------//

loadScript("https://www.lutanho.net/pgn/ltpgnviewer.js", function () {
    loaded.fen = true
    console.log("PgnToFen loaded")
});

//--------------------------------------STOCKFISH - CHESS ENGINE MODULE--------------------------------------//

loadScript("https://chess-bot.com/online_calculator/stockfish.js", function () {
    loadScript("https://chess-bot.com/online_calculator/stockfish.asm.js", function () {
        loaded.stockfish = true
        evaler = typeof STOCKFISH === "function" ? STOCKFISH() : new Worker(options.stockfishjs || 'stockfish.js');
        console.log("stockfish loaded")
        if (loaded.fen && loaded.stockfish) {
            evaler.onmessage = function (event) {
                var text;
                if (event && typeof event === "object") {
                    text = event.data;
                } else {
                    text = event;
                }
                if (debug) console.log("ANALYZE :", text);
                if (text !== undefined && text.startsWith("bestmove")) {
                    const move = text.split(" ")[1];
                    const peice = document.querySelector("#board-single > .square-" + MAPS.rank[move.slice(0, 1)] + move.slice(1, 2));
                    if (peices.from) {
                        peices.from.classList.remove("SelectedPeice");
                        peices.to.remove();
                    }
                    peices.from = peice;
                    peices.from.classList.add("SelectedPeice");
                    peices.to = peice.cloneNode(true);
                    peices.to.classList.remove("square-" + MAPS.rank[move.slice(0, 1)] + move.slice(1, 2))
                    console.log(move.slice(2, 3) + move.slice(3, 4))
                    peices.to.classList.add("square-" + MAPS.rank[move.slice(2, 3)] + move.slice(3, 4))
                    peices.to.style.opacity = "0.5";
                    peices.from.parentElement.appendChild(peices.to)
                    peices.to.onclick = () => {
                        peice.classList.remove("SelectedPeice");
                        peices.to.remove();
                    }
                    setTimeout(() => {
                        if (peice.classList.contains("SelectedPeice")) {
                            peice.classList.remove("SelectedPeice");
                            peices.to.remove();
                        }
                    }, 3000);
                };
            };
        };
    });
});
function show() {
    if (!evaler) return;
    evaler.postMessage("setoption name Hash value 256")
    evaler.postMessage("position fen " + getFen());
    evaler.postMessage("go movetime " + Math.floor(Math.random() * MAX_ANALYZE_TIME) * 1000);
}

const targetElement = document.querySelector('.clock-bottom');

const sleep = ms => new Promise(r => setTimeout(r, ms))
const observer = new MutationObserver(function (mutationsList) {
    mutationsList.forEach(async function (mutation) {
        if (mutation.type === 'attributes' && mutation.attributeName === 'class') {
            const classList = targetElement.classList;
            if (classList.contains('clock-player-turn')) {
                await sleep(Math.floor(Math.random() * MAX_DELAY) * 1000)
                show()
            }
        }
    });
});

observer.observe(targetElement, { attributes: true });
