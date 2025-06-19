<!DOCTYPE html>
<html lang="de">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Schachspiel</title>
  <style>
    body {
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      height: 100vh;
      background-color: #2e2e2e;
      margin: 0;
    }
    #chessboard {
      display: grid;
      grid-template-columns: repeat(8, 60px);
      grid-template-rows: repeat(8, 60px);
      border: 3px solid #fff;
    }
    .square {
      width: 60px;
      height: 60px;
      display: flex;
      justify-content: center;
      align-items: center;
      font-size: 36px;
      font-family: serif;
    }
    .white {
      background-color: #f0d9b5;
    }
    .black {
      background-color: #b58863;
    }
    .selected {
      outline: 3px solid yellow;
    }
    button {
      margin: 20px;
      padding: 10px 20px;
      font-size: 16px;
    }
  </style>
</head>
<body>
  <button onclick="undoMove()">Rückgängig</button>
  <div id="chessboard"></div>

  <script>
    const board = document.getElementById('chessboard');
    const initialBoard = [
      ['♜','♞','♝','♛','♚','♝','♞','♜'],
      ['♟','♟','♟','♟','♟','♟','♟','♟'],
      ['','','','','','','',''],
      ['','','','','','','',''],
      ['','','','','','','',''],
      ['','','','','','','',''],
      ['♙','♙','♙','♙','♙','♙','♙','♙'],
      ['♖','♘','♗','♕','♔','♗','♘','♖']
    ];

    let boardState = JSON.parse(JSON.stringify(initialBoard));
    let selected = null;
    let moveHistory = [];
    let whiteToMove = true;
    let castlingRights = {
      whiteKingMoved: false,
      whiteRookLeftMoved: false,
      whiteRookRightMoved: false,
      blackKingMoved: false,
      blackRookLeftMoved: false,
      blackRookRightMoved: false
    };

    // Für en passant: speichert die Spalte des Bauern, der gerade 2 Felder vorgerückt ist, sonst null
    let enPassantTarget = null;

    function renderBoard() {
      board.innerHTML = '';
      for (let row = 0; row < 8; row++) {
        for (let col = 0; col < 8; col++) {
          const square = document.createElement('div');
          square.classList.add('square');
          square.classList.add((row + col) % 2 === 0 ? 'white' : 'black');
          square.dataset.row = row;
          square.dataset.col = col;
          square.textContent = boardState[row][col];

          if (selected && selected.row == row && selected.col == col) {
            square.classList.add('selected');
          }

          square.addEventListener('click', handleClick);
          board.appendChild(square);
        }
      }
    }

    function isWhite(piece) {
      return ['♙','♖','♘','♗','♕','♔'].includes(piece);
    }

    function isBlack(piece) {
      return ['♟','♜','♞','♝','♛','♚'].includes(piece);
    }

    // Hilfsfunktion: Prüft ob Feld leer ist oder gegnerische Figur
    function isOpponent(piece, target) {
      if (target === '') return false;
      return (isWhite(piece) && isBlack(target)) || (isBlack(piece) && isWhite(target));
    }

    // Prüft ob Feld innerhalb des Bretts liegt
    function isInside(row, col) {
      return row >= 0 && row < 8 && col >= 0 && col < 8;
    }

    // Prüft, ob der König der angegebenen Farbe im Schach steht
    function isKingInCheck(white) {
      // König finden
      const king = white ? '♔' : '♚';
      let kingPos = null;
      for (let r = 0; r < 8; r++) {
        for (let c = 0; c < 8; c++) {
          if (boardState[r][c] === king) {
            kingPos = {row: r, col: c};
            break;
          }
        }
        if (kingPos) break;
      }
      if (!kingPos) return true; // König weg -> Schach

      // Gegnerische Figuren suchen, die den König bedrohen
      for (let r = 0; r < 8; r++) {
        for (let c = 0; c < 8; c++) {
          const p = boardState[r][c];
          if (p === '') continue;
          if (white && isBlack(p)) {
            if (canAttack(p, r, c, kingPos.row, kingPos.col)) return true;
          } else if (!white && isWhite(p)) {
            if (canAttack(p, r, c, kingPos.row, kingPos.col)) return true;
          }
        }
      }
      return false;
    }

    // Prüft ob eine Figur das Ziel angreifen kann, ohne Schachregeln wie "eigener König nicht in Schach" zu beachten
    function canAttack(piece, fromRow, fromCol, toRow, toCol) {
      const dx = toCol - fromCol;
      const dy = toRow - fromRow;
      const absDx = Math.abs(dx);
      const absDy = Math.abs(dy);
      const target = boardState[toRow][toCol];

      switch (piece) {
        case '♙': // Weißer Bauer greift diagonal nach vorne
          return dy === -1 && absDx === 1 && isBlack(target);
        case '♟': // Schwarzer Bauer greift diagonal nach vorne
          return dy === 1 && absDx === 1 && isWhite(target);
        case '♖':
        case '♜':
          if (dx !== 0 && dy !== 0) return false;
          return clearLine(fromRow, fromCol, toRow, toCol);
        case '♗':
        case '♝':
          if (absDx !== absDy) return false;
          return clearLine(fromRow, fromCol, toRow, toCol);
        case '♘':
        case '♞':
          return (absDx === 2 && absDy === 1) || (absDx === 1 && absDy === 2);
        case '♕':
        case '♛':
          if (dx === 0 || dy === 0 || absDx === absDy) {
            return clearLine(fromRow, fromCol, toRow, toCol);
          }
          return false;
        case '♔':
        case '♚':
          return absDx <= 1 && absDy <= 1;
      }
      return false;
    }

    // Prüft, ob alle Felder zwischen from und to frei sind (ohne from und to)
    function clearLine(fromRow, fromCol, toRow, toCol) {
      let stepRow = Math.sign(toRow - fromRow);
      let stepCol = Math.sign(toCol - fromCol);
      let currentRow = fromRow + stepRow;
      let currentCol = fromCol + stepCol;

      while (currentRow !== toRow || currentCol !== toCol) {
        if (boardState[currentRow][currentCol] !== '') return false;
        currentRow += stepRow;
        currentCol += stepCol;
      }
      return true;
    }

    // Prüft ob ein Zug legal ist (inkl. Schach-Regel: eigener König darf nicht ins Schach)
    function isLegalMove(piece, fromRow, fromCol, toRow, toCol) {
      if (piece === '') return false;

      // Keine Züge außerhalb des Brettes
      if (!isInside(toRow, toCol)) return false;

      const target = boardState[toRow][toCol];
      if ((isWhite(piece) && isWhite(target)) || (isBlack(piece) && isBlack(target))) return false;

      const dx = toCol - fromCol;
      const dy = toRow - fromRow;
      const absDx = Math.abs(dx);
      const absDy = Math.abs(dy);

      // Speichern des aktuellen States, um es ggf. zurückzusetzen
      const backupBoard = JSON.parse(JSON.stringify(boardState));
      const backupCastling = JSON.parse(JSON.stringify(castlingRights));
      const backupEnPassant = enPassantTarget;
      const backupWhiteToMove = whiteToMove;

      // Probeweise Zug ausführen, um Schach zu prüfen
      function makeTestMove() {
        // Rochade
        if (piece === '♔' && fromRow === 7 && fromCol === 4 && toRow === 7 && (toCol === 6 || toCol === 2)) {
          // Kurze Rochade
          if (toCol === 6) {
            boardState[7][4] = '';
            boardState[7][6] = '♔';
            boardState[7][7] = '';
            boardState[7][5] = '♖';
          } else { // Lange Rochade
            boardState[7][4] = '';
            boardState[7][2] = '♔';
            boardState[7][0] = '';
            boardState[7][3] = '♖';
          }
        } else if (piece === '♚' && fromRow === 0 && fromCol === 4 && toRow === 0 && (toCol === 6 || toCol === 2)) {
          if (toCol === 6) {
            boardState[0][4] = '';
            boardState[0][6] = '♚';
            boardState[0][7] = '';
            boardState[0][5] = '♜';
          } else {
            boardState[0][4] = '';
            boardState[0][2] = '♚';
            boardState[0][0] = '';
            boardState[0][3] = '♜';
          }
        } else {
          // Normales Ziehen oder en passant
          boardState[toRow][toCol] = piece;
          boardState[fromRow][fromCol] = '';

          // En passant: falls Bauer diagonal auf leer Feld zieht und enPassantTarget stimmt
          if ((piece === '♙' || piece === '♟') && toCol !== fromCol && target === '') {
            // Schlage Bauern "en passant"
            const dir = piece === '♙' ? 1 : -1;
            boardState[toRow + dir][toCol] = '';
          }
        }
      }

      makeTestMove();

      // Prüfe ob König nach Zug im Schach steht
      const kingInCheck = whiteToMove ? isKingInCheck(true) : isKingInCheck(false);

      // Zustand wiederherstellen
      boardState = backupBoard;
      castlingRights = backupCastling;
      enPassantTarget = backupEnPassant;
      whiteToMove = backupWhiteToMove;

      if (kingInCheck) return false;

      // Validierung je Figur
      switch (piece) {
        case '♙': // Weißer Bauer
          // Vorwärts
          if (dx === 0 && dy === -1 && target === '') return true;
          // Erster Zug 2 Felder
          if (dx === 0 && dy === -2 && fromRow === 6 && target === '' && boardState[fromRow - 1][fromCol] === '') return true;
          // Schlagen diagonal
          if (absDx === 1 && dy === -1 && (isBlack(target) || (enPassantTarget === toCol && fromRow === 3))) return true;
          return false;

        case '♟': // Schwarzer Bauer
          if (dx === 0 && dy === 1 && target === '') return true;
          if (dx === 0 && dy === 2 && fromRow === 1 && target === '' && boardState[fromRow + 1][fromCol] === '') return true;
          if (absDx === 1 && dy === 1 && (isWhite(target) || (enPassantTarget === toCol && fromRow === 4))) return true;
          return false;

        case '♖': case '♜': // Turm
          if (dx !== 0 && dy !== 0) return false;
          return clearLine(fromRow, fromCol, toRow, toCol);

        case '♗': case '♝': // Läufer
          if (absDx !== absDy) return false;
          return clearLine(fromRow, fromCol, toRow, toCol);

        case '♘': case '♞': // Springer
          return (absDx === 2 && absDy === 1) || (absDx === 1 && absDy === 2);

        case '♕': case '♛': // Dame
          if (dx === 0 || dy === 0 || absDx === absDy) {
            return clearLine(fromRow, fromCol, toRow, toCol);
          }
          return false;

        case '♔': case '♚': // König
          // Normaler Königszug 1 Feld in jede Richtung
          if (absDx <= 1 && absDy <= 1) return true;

          // Rochade prüfen
          if (piece === '♔' && fromRow === 7 && fromCol === 4) {
            // Kurze Rochade weiß
            if (toRow === 7 && toCol === 6) {
              if (castlingRights.whiteKingMoved || castlingRights.whiteRookRightMoved) return false;
              if (boardState[7][5] !== '' || boardState[7][6] !== '') return false;
              if (isKingInCheck(true)) return false;
              // Zwischenfelder nicht unter Angriff
              // Probeweise König auf Zwischenfelder setzen und prüfen
              boardState[7][4] = '';
              boardState[7][5] = '♔';
              if (isKingInCheck(true)) {
                boardState = backupBoard;
                return false;
              }
              boardState[7][5] = '';
              boardState[7][6] = '♔';
              if (isKingInCheck(true)) {
                boardState = backupBoard;
                return false;
              }
              boardState = backupBoard;
              return true;
            }
            // Lange Rochade weiß
            if (toRow === 7 && toCol === 2) {
              if (castlingRights.whiteKingMoved || castlingRights.whiteRookLeftMoved) return false;
              if (boardState[7][1] !== '' || boardState[7][2] !== '' || boardState[7][3] !== '') return false;
              if (isKingInCheck(true)) return false;
              boardState[7][4] = '';
              boardState[7][3] = '♔';
              if (isKingInCheck(true)) {
                boardState = backupBoard;
                return false;
              }
              boardState[7][3] = '';
              boardState[7][2] = '♔';
              if (isKingInCheck(true)) {
                boardState = backupBoard;
                return false;
              }
              boardState = backupBoard;
              return true;
            }
          }
          if (piece === '♚' && fromRow === 0 && fromCol === 4) {
            // Kurze Rochade schwarz
            if (toRow === 0 && toCol === 6) {
              if (castlingRights.blackKingMoved || castlingRights.blackRookRightMoved) return false;
              if (boardState[0][5] !== '' || boardState[0][6] !== '') return false;
              if (isKingInCheck(false)) return false;
              boardState[0][4] = '';
              boardState[0][5] = '♚';
              if (isKingInCheck(false)) {
                boardState = backupBoard;
                return false;
              }
              boardState[0][5] = '';
              boardState[0][6] = '♚';
              if (isKingInCheck(false)) {
                boardState = backupBoard;
                return false;
              }
              boardState = backupBoard;
              return true;
            }
            // Lange Rochade schwarz
            if (toRow === 0 && toCol === 2) {
              if (castlingRights.blackKingMoved || castlingRights.blackRookLeftMoved) return false;
              if (boardState[0][1] !== '' || boardState[0][2] !== '' || boardState[0][3] !== '') return false;
              if (isKingInCheck(false)) return false;
              boardState[0][4] = '';
              boardState[0][3] = '♚';
              if (isKingInCheck(false)) {
                boardState = backupBoard;
                return false;
              }
              boardState[0][3] = '';
              boardState[0][2] = '♚';
              if (isKingInCheck(false)) {
                boardState = backupBoard;
                return false;
              }
              boardState = backupBoard;
              return true;
            }
          }
          return false;
      }
      return false;
    }

    function handleClick(e) {
      const row = parseInt(e.target.dataset.row);
      const col = parseInt(e.target.dataset.col);
      const clickedPiece = boardState[row][col];

      if (selected) {
        // Versuch Zug
        const fromRow = selected.row;
        const fromCol = selected.col;
        const piece = boardState[fromRow][fromCol];

        if ((whiteToMove && !isWhite(piece)) || (!whiteToMove && !isBlack(piece))) {
          selected = null;
          renderBoard();
          return;
        }

        if (isLegalMove(piece, fromRow, fromCol, row, col)) {
          // Speichere aktuellen Zustand für Rückgängig
          moveHistory.push({
            board: JSON.parse(JSON.stringify(boardState)),
            turn: whiteToMove,
            castling: JSON.parse(JSON.stringify(castlingRights)),
            enPassant: enPassantTarget
          });

          // Zug ausführen

          // Rochade
          if ((piece === '♔' || piece === '♚') && Math.abs(col - fromCol) === 2) {
            if (col === 6) { // kurze Rochade
              boardState[row][fromCol] = '';
              boardState[row][col] = piece;
              boardState[row][7] = '';
              boardState[row][5] = (piece === '♔') ? '♖' : '♜';
            } else if (col === 2) { // lange Rochade
              boardState[row][fromCol] = '';
              boardState[row][col] = piece;
              boardState[row][0] = '';
              boardState[row][3] = (piece === '♔') ? '♖' : '♜';
            }
            // Rochaderechte anpassen
            if (piece === '♔') {
              castlingRights.whiteKingMoved = true;
              castlingRights.whiteRookLeftMoved = true;
              castlingRights.whiteRookRightMoved = true;
            } else {
              castlingRights.blackKingMoved = true;
              castlingRights.blackRookLeftMoved = true;
              castlingRights.blackRookRightMoved = true;
            }
          } else {
            // En passant schlagen
            if ((piece === '♙' || piece === '♟') && col !== fromCol && boardState[row][col] === '') {
              // Bauer schlägt en passant
              const dir = piece === '♙' ? 1 : -1;
              boardState[row + dir][col] = '';
            }

            boardState[row][col] = piece;
            boardState[fromRow][fromCol] = '';
          }

          // Rochaderechte anpassen bei König oder Turmzug
          if (piece === '♔') castlingRights.whiteKingMoved = true;
          if (piece === '♚') castlingRights.blackKingMoved = true;
          if (piece === '♖' && fromRow === 7 && fromCol === 0) castlingRights.whiteRookLeftMoved = true;
          if (piece === '♖' && fromRow === 7 && fromCol === 7) castlingRights.whiteRookRightMoved = true;
          if (piece === '♜' && fromRow === 0 && fromCol === 0) castlingRights.blackRookLeftMoved = true;
          if (piece === '♜' && fromRow === 0 && fromCol === 7) castlingRights.blackRookRightMoved = true;

          // En passant Ziel setzen
          enPassantTarget = null;
          if (piece === '♙' && fromRow === 6 && row === 4) enPassantTarget = col;
          if (piece === '♟' && fromRow === 1 && row === 3) enPassantTarget = col;

          // Bauernumwandlung (Promotion)
          if (piece === '♙' && row === 0) {
            // Einfach Dame automatisch (kann man erweitern)
            boardState[row][col] = '♕';
          }
          if (piece === '♟' && row === 7) {
            boardState[row][col] = '♛';
          }

          whiteToMove = !whiteToMove;
        }

        selected = null;
        renderBoard();
      } else {
        // Keine Figur ausgewählt -> nur eigene Figuren auswählen
        if ((whiteToMove && isWhite(clickedPiece)) || (!whiteToMove && isBlack(clickedPiece))) {
          selected = {row, col};
          renderBoard();
        }
      }
    }

    function undoMove() {
      if (moveHistory.length > 0) {
        const last = moveHistory.pop();
        boardState = last.board;
        whiteToMove = last.turn;
        castlingRights = last.castling;
        enPassantTarget = last.enPassant;
        renderBoard();
      }
    }

    renderBoard();
  </script>
</body>
</html>
