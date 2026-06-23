# Machine Coding: Chess Game

---

## 1. Problem Statement (as given in a real interview)

> Design and implement a two-player Chess game in Java. The game should support a standard 8x8 board with all six piece types (King, Queen, Rook, Bishop, Knight, Pawn). Players take turns entering moves. The system must validate that every move is legal for the piece type, detect when a player is in check, and end the game when checkmate or stalemate is reached.
>
> You have 90 minutes. Focus on correctness of piece movement, turn management, check detection, and clean object-oriented design. You may treat the UI as a simple console and ignore en passant, castling, and pawn promotion for now — but be prepared to discuss how you would add them.

---

## 2. Functional Requirements

| # | Requirement |
|---|-------------|
| FR-1 | The game is played on an 8x8 board. Rows are numbered 0–7 (rank 1–8), columns 0–7 (file a–h). |
| FR-2 | Two players: WHITE moves first. Players alternate turns. |
| FR-3 | Each of the six piece types (King, Queen, Rook, Bishop, Knight, Pawn) has its own movement rules. |
| FR-4 | A move is illegal if: the destination is occupied by a friendly piece, the path is blocked (for sliding pieces), or the move leaves the moving player's own King in check. |
| FR-5 | After every move, detect if the opponent is now in check. |
| FR-6 | Detect checkmate: the player is in check and has no legal move. The game ends; the opponent wins. |
| FR-7 | Detect stalemate: the player is not in check but has no legal move. The game ends in a draw. |
| FR-8 | The game tracks its state: ACTIVE, WHITE_WINS, BLACK_WINS, STALEMATE. |

**Out of scope (mention as extensions):** En passant, castling, pawn promotion, draw by 50-move rule, draw by repetition, timed games.

---

## 3. High-Level Design

### 3.1 Entity Identification

| Entity | Responsibility |
|--------|---------------|
| `Board` | Owns the 8×8 grid of `Cell` objects; knows where every piece is; provides helper queries (path-clear checks, find-king). |
| `Cell` | Value object: `(row, col)` + optional reference to the `Piece` occupying it. |
| `Piece` | Abstract base: colour, type; declares `isValidMove(Board, Cell, Cell)`. |
| `King / Queen / Rook / Bishop / Knight / Pawn` | Concrete subclasses; each implements its own movement logic. |
| `Move` | Record of a single half-move: player, from-cell, to-cell, piece moved, piece captured (nullable). |
| `Player` | Name + colour. |
| `Game` | Orchestrates the game loop: validates moves, applies them, updates `GameStatus`, detects check/checkmate/stalemate. |
| `GameStatus` (enum) | `ACTIVE`, `WHITE_WINS`, `BLACK_WINS`, `STALEMATE`. |
| `PieceColor` (enum) | `WHITE`, `BLACK`. |
| `PieceType` (enum) | `KING`, `QUEEN`, `ROOK`, `BISHOP`, `KNIGHT`, `PAWN`. |

### 3.2 Key Design Decisions

1. **Move validation is delegated to the piece (Strategy pattern).** Each piece knows its own movement rules. `Game` calls `piece.isValidMove()` and then applies a second layer check: "does this move leave my King in check?"

2. **Simulate-and-test for check safety.** Rather than writing complex pin-detection logic, the `Game` class uses a *simulate-and-revert* approach: apply the move on a copy of the board, check if own King is in check, then revert. This is correct and simple to implement in 90 minutes.

3. **`Board` is mutable; moves are applied directly.** A separate `applyMove` / `undoMove` pair keeps the board consistent. This avoids deep-copying the entire board on every legality check.

4. **Check detection is `isInCheck(PieceColor, Board)`.** It iterates every opponent piece and asks whether it can attack the King's cell. This reuses existing `isValidMove` logic.

5. **Checkmate / stalemate detection** collects every possible move for the current player and filters for legal ones (those that do not leave the King in check). If the set is empty, check status determines checkmate vs stalemate.

### 3.3 Class Diagram (textual)

```
Game
 ├── Board
 │    └── Cell[8][8]
 │         └── Piece (nullable)
 ├── Player (white, black)
 ├── GameStatus (enum)
 └── List<Move> (history)

Piece  <<abstract>>
 ├── King
 ├── Queen
 ├── Rook
 ├── Bishop
 ├── Knight
 └── Pawn
```

---

## 4. Complete Java Implementation

### 4.1 Enums

```java
// PieceColor.java
public enum PieceColor {
    WHITE, BLACK;

    public PieceColor opposite() {
        return this == WHITE ? BLACK : WHITE;
    }
}
```

```java
// PieceType.java
public enum PieceType {
    KING, QUEEN, ROOK, BISHOP, KNIGHT, PAWN
}
```

```java
// GameStatus.java
public enum GameStatus {
    ACTIVE, WHITE_WINS, BLACK_WINS, STALEMATE
}
```

---

### 4.2 Cell

```java
// Cell.java
public class Cell {
    private final int row;   // 0 = rank 1 (White's back rank), 7 = rank 8
    private final int col;   // 0 = file a, 7 = file h
    private Piece piece;

    public Cell(int row, int col) {
        this.row = row;
        this.col = col;
        this.piece = null;
    }

    // Copy constructor (used for board simulation)
    public Cell(Cell other) {
        this.row = other.row;
        this.col = other.col;
        this.piece = other.piece; // Piece reference shared — fine for read-only simulation
    }

    public int getRow() { return row; }
    public int getCol() { return col; }

    public Piece getPiece() { return piece; }
    public void setPiece(Piece piece) { this.piece = piece; }

    public boolean isEmpty() { return piece == null; }

    public boolean isOccupiedByColor(PieceColor color) {
        return piece != null && piece.getColor() == color;
    }

    @Override
    public String toString() {
        char file = (char) ('a' + col);
        int rank = row + 1;
        return "" + file + rank;
    }
}
```

---

### 4.3 Piece (abstract base)

```java
// Piece.java
public abstract class Piece {
    protected final PieceColor color;
    protected final PieceType type;

    public Piece(PieceColor color, PieceType type) {
        this.color = color;
        this.type = type;
    }

    public PieceColor getColor() { return color; }
    public PieceType getType()   { return type;  }

    /**
     * Returns true if moving this piece from 'from' to 'to' is geometrically
     * legal for this piece type, ignoring whether the move exposes own King to check.
     * The Board reference is needed by sliding pieces to detect blocked paths.
     */
    public abstract boolean isValidMove(Board board, Cell from, Cell to);

    @Override
    public String toString() {
        return color.name().charAt(0) + "" + type.name().charAt(0);
    }
}
```

---

### 4.4 Piece Subclasses

#### King

```java
// King.java
public class King extends Piece {

    public King(PieceColor color) {
        super(color, PieceType.KING);
    }

    @Override
    public boolean isValidMove(Board board, Cell from, Cell to) {
        // Cannot capture own piece
        if (to.isOccupiedByColor(color)) return false;

        int rowDiff = Math.abs(to.getRow() - from.getRow());
        int colDiff = Math.abs(to.getCol() - from.getCol());

        // King moves exactly one square in any direction
        return rowDiff <= 1 && colDiff <= 1 && (rowDiff + colDiff) > 0;
    }
}
```

#### Queen

```java
// Queen.java
public class Queen extends Piece {

    public Queen(PieceColor color) {
        super(color, PieceType.QUEEN);
    }

    @Override
    public boolean isValidMove(Board board, Cell from, Cell to) {
        if (to.isOccupiedByColor(color)) return false;

        int rowDiff = Math.abs(to.getRow() - from.getRow());
        int colDiff = Math.abs(to.getCol() - from.getCol());

        // Queen = Rook OR Bishop movement
        boolean straightLine = (from.getRow() == to.getRow() || from.getCol() == to.getCol());
        boolean diagonal     = (rowDiff == colDiff);

        if (!straightLine && !diagonal) return false;

        // Path must be clear
        return board.isPathClear(from, to);
    }
}
```

#### Rook

```java
// Rook.java
public class Rook extends Piece {

    public Rook(PieceColor color) {
        super(color, PieceType.ROOK);
    }

    @Override
    public boolean isValidMove(Board board, Cell from, Cell to) {
        if (to.isOccupiedByColor(color)) return false;

        // Must move along a rank or file
        if (from.getRow() != to.getRow() && from.getCol() != to.getCol()) return false;

        return board.isPathClear(from, to);
    }
}
```

#### Bishop

```java
// Bishop.java
public class Bishop extends Piece {

    public Bishop(PieceColor color) {
        super(color, PieceType.BISHOP);
    }

    @Override
    public boolean isValidMove(Board board, Cell from, Cell to) {
        if (to.isOccupiedByColor(color)) return false;

        int rowDiff = Math.abs(to.getRow() - from.getRow());
        int colDiff = Math.abs(to.getCol() - from.getCol());

        // Must be a diagonal (equal row and col delta, non-zero)
        if (rowDiff != colDiff || rowDiff == 0) return false;

        return board.isPathClear(from, to);
    }
}
```

#### Knight

```java
// Knight.java
public class Knight extends Piece {

    public Knight(PieceColor color) {
        super(color, PieceType.KNIGHT);
    }

    @Override
    public boolean isValidMove(Board board, Cell from, Cell to) {
        if (to.isOccupiedByColor(color)) return false;

        int rowDiff = Math.abs(to.getRow() - from.getRow());
        int colDiff = Math.abs(to.getCol() - from.getCol());

        // L-shape: (2,1) or (1,2)
        return (rowDiff == 2 && colDiff == 1) || (rowDiff == 1 && colDiff == 2);
        // Knights jump — no path check needed
    }
}
```

#### Pawn

```java
// Pawn.java
public class Pawn extends Piece {

    public Pawn(PieceColor color) {
        super(color, PieceType.PAWN);
    }

    @Override
    public boolean isValidMove(Board board, Cell from, Cell to) {
        // Cannot capture own piece
        if (to.isOccupiedByColor(color)) return false;

        // White pawns advance from row 0 toward row 7 (increasing row index).
        // Black pawns advance from row 7 toward row 0 (decreasing row index).
        int direction = (color == PieceColor.WHITE) ? 1 : -1;
        int startRow  = (color == PieceColor.WHITE) ? 1 : 6;

        int rowDiff = to.getRow() - from.getRow();
        int colDiff = to.getCol() - from.getCol();

        // --- Forward move (no capture) ---
        if (colDiff == 0) {
            if (rowDiff == direction) {
                // Single step: destination must be empty
                return to.isEmpty();
            }
            if (rowDiff == 2 * direction && from.getRow() == startRow) {
                // Double step from starting row: both intermediate and destination must be empty
                Cell intermediate = board.getCell(from.getRow() + direction, from.getCol());
                return intermediate.isEmpty() && to.isEmpty();
            }
            return false;
        }

        // --- Diagonal capture ---
        if (Math.abs(colDiff) == 1 && rowDiff == direction) {
            // Must capture an opponent's piece
            return !to.isEmpty() && to.getPiece().getColor() != color;
        }

        return false;
    }
}
```

---

### 4.5 Board

```java
// Board.java
import java.util.ArrayList;
import java.util.List;

public class Board {

    private final Cell[][] grid;  // grid[row][col]

    public Board() {
        grid = new Cell[8][8];
        for (int r = 0; r < 8; r++) {
            for (int c = 0; c < 8; c++) {
                grid[r][c] = new Cell(r, c);
            }
        }
        initializePieces();
    }

    /** Deep-copy constructor used for move simulation. */
    public Board(Board other) {
        grid = new Cell[8][8];
        for (int r = 0; r < 8; r++) {
            for (int c = 0; c < 8; c++) {
                grid[r][c] = new Cell(other.grid[r][c]);
            }
        }
    }

    // -------------------------------------------------------------------------
    // Initialisation
    // -------------------------------------------------------------------------

    private void initializePieces() {
        // White pieces on ranks 0 and 1 (rows 0 and 1)
        placePiecesForColor(PieceColor.WHITE, 0, 1);
        // Black pieces on ranks 7 and 8 (rows 7 and 6)
        placePiecesForColor(PieceColor.BLACK, 7, 6);
    }

    private void placePiecesForColor(PieceColor color, int backRow, int pawnRow) {
        grid[backRow][0].setPiece(new Rook(color));
        grid[backRow][1].setPiece(new Knight(color));
        grid[backRow][2].setPiece(new Bishop(color));
        grid[backRow][3].setPiece(new Queen(color));
        grid[backRow][4].setPiece(new King(color));
        grid[backRow][5].setPiece(new Bishop(color));
        grid[backRow][6].setPiece(new Knight(color));
        grid[backRow][7].setPiece(new Rook(color));

        for (int c = 0; c < 8; c++) {
            grid[pawnRow][c].setPiece(new Pawn(color));
        }
    }

    // -------------------------------------------------------------------------
    // Cell access
    // -------------------------------------------------------------------------

    public Cell getCell(int row, int col) {
        if (!isInBounds(row, col)) {
            throw new IllegalArgumentException("Cell out of bounds: " + row + "," + col);
        }
        return grid[row][col];
    }

    public boolean isInBounds(int row, int col) {
        return row >= 0 && row < 8 && col >= 0 && col < 8;
    }

    // -------------------------------------------------------------------------
    // Path clearance (for sliding pieces: Queen, Rook, Bishop)
    // -------------------------------------------------------------------------

    /**
     * Returns true if every intermediate cell between 'from' and 'to' is empty.
     * Does NOT check the destination cell (that is handled separately by each piece).
     * Precondition: the caller has already verified that the move is straight or diagonal.
     */
    public boolean isPathClear(Cell from, Cell to) {
        int rowStep = Integer.signum(to.getRow() - from.getRow());
        int colStep = Integer.signum(to.getCol() - from.getCol());

        int row = from.getRow() + rowStep;
        int col = from.getCol() + colStep;

        while (row != to.getRow() || col != to.getCol()) {
            if (!grid[row][col].isEmpty()) return false;
            row += rowStep;
            col += colStep;
        }
        return true;
    }

    // -------------------------------------------------------------------------
    // King location
    // -------------------------------------------------------------------------

    public Cell findKing(PieceColor color) {
        for (int r = 0; r < 8; r++) {
            for (int c = 0; c < 8; c++) {
                Piece p = grid[r][c].getPiece();
                if (p != null && p.getType() == PieceType.KING && p.getColor() == color) {
                    return grid[r][c];
                }
            }
        }
        throw new IllegalStateException("King not found for color: " + color);
    }

    // -------------------------------------------------------------------------
    // All cells occupied by a given color
    // -------------------------------------------------------------------------

    public List<Cell> getCellsOccupiedBy(PieceColor color) {
        List<Cell> cells = new ArrayList<>();
        for (int r = 0; r < 8; r++) {
            for (int c = 0; c < 8; c++) {
                if (grid[r][c].isOccupiedByColor(color)) {
                    cells.add(grid[r][c]);
                }
            }
        }
        return cells;
    }

    // -------------------------------------------------------------------------
    // Move application / undo (used by Game for real moves and simulations)
    // -------------------------------------------------------------------------

    /**
     * Applies a move on this board (mutates in place).
     * Returns the piece that was captured (null if none) so the caller can undo.
     */
    public Piece applyMove(Cell from, Cell to) {
        Piece captured = to.getPiece();
        to.setPiece(from.getPiece());
        from.setPiece(null);
        return captured;
    }

    /**
     * Reverts a previously applied move.
     */
    public void undoMove(Cell from, Cell to, Piece capturedPiece, Piece movedPiece) {
        from.setPiece(movedPiece);
        to.setPiece(capturedPiece);
    }

    // -------------------------------------------------------------------------
    // Display
    // -------------------------------------------------------------------------

    public void print() {
        System.out.println("  a  b  c  d  e  f  g  h");
        for (int r = 7; r >= 0; r--) {
            System.out.print((r + 1) + " ");
            for (int c = 0; c < 8; c++) {
                Piece p = grid[r][c].getPiece();
                if (p == null) {
                    System.out.print(" . ");
                } else {
                    System.out.print(p + " ");
                }
            }
            System.out.println(r + 1);
        }
        System.out.println("  a  b  c  d  e  f  g  h");
    }
}
```

---

### 4.6 Move

```java
// Move.java
public class Move {
    private final Player player;
    private final Cell from;
    private final Cell to;
    private final Piece pieceMoved;
    private final Piece pieceCapured;  // null if no capture

    public Move(Player player, Cell from, Cell to, Piece pieceMoved, Piece pieceCaptured) {
        this.player       = player;
        this.from         = from;
        this.to           = to;
        this.pieceMoved   = pieceMoved;
        this.pieceCapured = pieceCaptured;
    }

    public Player getPlayer()        { return player; }
    public Cell getFrom()            { return from; }
    public Cell getTo()              { return to; }
    public Piece getPieceMoved()     { return pieceMoved; }
    public Piece getPieceCaptured()  { return pieceCapured; }

    @Override
    public String toString() {
        return player.getName() + ": " + pieceMoved + " " + from + " -> " + to
                + (pieceCapured != null ? " captures " + pieceCapured : "");
    }
}
```

---

### 4.7 Player

```java
// Player.java
public class Player {
    private final String name;
    private final PieceColor color;

    public Player(String name, PieceColor color) {
        this.name  = name;
        this.color = color;
    }

    public String getName()    { return name;  }
    public PieceColor getColor() { return color; }

    @Override
    public String toString() {
        return name + " (" + color + ")";
    }
}
```

---

### 4.8 Game

```java
// Game.java
import java.util.ArrayList;
import java.util.List;

public class Game {

    private final Board board;
    private final Player whitePlayer;
    private final Player blackPlayer;
    private Player currentPlayer;
    private GameStatus status;
    private final List<Move> moveHistory;

    public Game(Player whitePlayer, Player blackPlayer) {
        this.board       = new Board();
        this.whitePlayer = whitePlayer;
        this.blackPlayer = blackPlayer;
        this.currentPlayer = whitePlayer;  // White moves first
        this.status      = GameStatus.ACTIVE;
        this.moveHistory = new ArrayList<>();
    }

    // -------------------------------------------------------------------------
    // Public API: make a move
    // -------------------------------------------------------------------------

    /**
     * Attempts to execute a move from (fromRow, fromCol) to (toRow, toCol)
     * for the current player.
     *
     * @return true if the move was accepted; false if it was illegal.
     */
    public boolean makeMove(int fromRow, int fromCol, int toRow, int toCol) {
        if (status != GameStatus.ACTIVE) {
            System.out.println("Game is already over: " + status);
            return false;
        }

        Cell from = board.getCell(fromRow, fromCol);
        Cell to   = board.getCell(toRow, toCol);

        // --- Basic sanity checks ---
        if (from.isEmpty()) {
            System.out.println("No piece at " + from);
            return false;
        }
        Piece piece = from.getPiece();
        if (piece.getColor() != currentPlayer.getColor()) {
            System.out.println("That is not your piece.");
            return false;
        }

        // --- Piece-specific geometric legality ---
        if (!piece.isValidMove(board, from, to)) {
            System.out.println("Illegal move for " + piece.getType());
            return false;
        }

        // --- Check that the move does not leave own King in check ---
        if (moveLeavesKingInCheck(from, to, currentPlayer.getColor())) {
            System.out.println("That move leaves your King in check!");
            return false;
        }

        // --- Apply the move ---
        Piece captured = board.applyMove(from, to);
        Move move = new Move(currentPlayer, from, to, piece, captured);
        moveHistory.add(move);
        System.out.println(move);

        // --- Update game status ---
        Player opponent = (currentPlayer == whitePlayer) ? blackPlayer : whitePlayer;
        updateStatus(opponent);

        // --- Switch turns if game is still active ---
        if (status == GameStatus.ACTIVE) {
            currentPlayer = opponent;
        }

        return true;
    }

    // -------------------------------------------------------------------------
    // Status update after each move
    // -------------------------------------------------------------------------

    private void updateStatus(Player opponent) {
        boolean opponentInCheck    = isInCheck(opponent.getColor());
        boolean opponentHasMoves   = hasAnyLegalMove(opponent.getColor());

        if (!opponentHasMoves) {
            if (opponentInCheck) {
                // Checkmate
                status = (opponent.getColor() == PieceColor.BLACK)
                        ? GameStatus.WHITE_WINS
                        : GameStatus.BLACK_WINS;
                System.out.println("Checkmate! " + currentPlayer + " wins!");
            } else {
                // Stalemate
                status = GameStatus.STALEMATE;
                System.out.println("Stalemate! Game is a draw.");
            }
        } else if (opponentInCheck) {
            System.out.println(opponent.getName() + " is in check!");
        }
    }

    // -------------------------------------------------------------------------
    // Check detection
    // -------------------------------------------------------------------------

    /**
     * Returns true if the King of the given color is currently under attack
     * by any opponent piece on the board.
     */
    public boolean isInCheck(PieceColor color) {
        Cell kingCell = board.findKing(color);
        PieceColor opponentColor = color.opposite();

        for (Cell attackerCell : board.getCellsOccupiedBy(opponentColor)) {
            Piece attacker = attackerCell.getPiece();
            if (attacker.isValidMove(board, attackerCell, kingCell)) {
                return true;
            }
        }
        return false;
    }

    // -------------------------------------------------------------------------
    // Checkmate / stalemate helper: does the color have ANY legal move?
    // -------------------------------------------------------------------------

    /**
     * Returns true if the given color has at least one legal move.
     * A move is legal if:
     *   (a) the piece's isValidMove returns true, AND
     *   (b) the move does not leave own King in check.
     */
    public boolean hasAnyLegalMove(PieceColor color) {
        for (Cell from : board.getCellsOccupiedBy(color)) {
            for (int r = 0; r < 8; r++) {
                for (int c = 0; c < 8; c++) {
                    Cell to = board.getCell(r, c);
                    Piece piece = from.getPiece();
                    if (piece.isValidMove(board, from, to)
                            && !moveLeavesKingInCheck(from, to, color)) {
                        return true;
                    }
                }
            }
        }
        return false;
    }

    // -------------------------------------------------------------------------
    // Simulate-and-revert: does this move expose own King to check?
    // -------------------------------------------------------------------------

    /**
     * Temporarily applies the move on the live board, tests for check,
     * then reverts. This avoids cloning the entire board on every call.
     */
    private boolean moveLeavesKingInCheck(Cell from, Cell to, PieceColor color) {
        Piece movedPiece    = from.getPiece();
        Piece capturedPiece = board.applyMove(from, to);

        boolean inCheck = isInCheck(color);

        board.undoMove(from, to, capturedPiece, movedPiece);
        return inCheck;
    }

    // -------------------------------------------------------------------------
    // Getters
    // -------------------------------------------------------------------------

    public GameStatus getStatus()       { return status; }
    public Player getCurrentPlayer()    { return currentPlayer; }
    public Board getBoard()             { return board; }
    public List<Move> getMoveHistory()  { return List.copyOf(moveHistory); }

    public boolean isOver() {
        return status != GameStatus.ACTIVE;
    }
}
```

---

### 4.9 Main (console game loop)

```java
// Main.java
import java.util.Scanner;

public class Main {

    public static void main(String[] args) {
        Player white = new Player("Alice", PieceColor.WHITE);
        Player black = new Player("Bob",   PieceColor.BLACK);
        Game game = new Game(white, black);

        Scanner scanner = new Scanner(System.in);

        System.out.println("Chess Game Started!");
        System.out.println("Enter moves as: fromRow fromCol toRow toCol  (0-indexed)");
        System.out.println("Example: 1 4 3 4  (moves the pawn on e2 to e4)");
        System.out.println();

        while (!game.isOver()) {
            game.getBoard().print();
            System.out.println();
            System.out.print(game.getCurrentPlayer() + " > ");

            if (!scanner.hasNextInt()) break;
            int fromRow = scanner.nextInt();
            int fromCol = scanner.nextInt();
            int toRow   = scanner.nextInt();
            int toCol   = scanner.nextInt();

            game.makeMove(fromRow, fromCol, toRow, toCol);
            System.out.println();
        }

        game.getBoard().print();
        System.out.println("Game over: " + game.getStatus());
        scanner.close();
    }
}
```

---

### 4.10 Complete file listing

```
chess/
├── PieceColor.java
├── PieceType.java
├── GameStatus.java
├── Cell.java
├── Piece.java
├── King.java
├── Queen.java
├── Rook.java
├── Bishop.java
├── Knight.java
├── Pawn.java
├── Board.java
├── Move.java
├── Player.java
├── Game.java
└── Main.java
```

---

## 5. Design Patterns Used

### 5.1 Strategy Pattern — Move Validation

**Where:** `Piece.isValidMove(Board, Cell, Cell)` is an abstract method; each subclass provides its own strategy.

**Why:** Chess has six fundamentally different movement rules. Without Strategy, `Game` would need a giant `if-else` or `switch` keyed on piece type. Adding a new piece type (e.g., a fairy chess piece) requires only a new subclass — `Game` and `Board` are untouched (Open/Closed Principle).

```
«interface / abstract»
       Piece
  isValidMove(board, from, to) : boolean
        |
  +---------+---------+----------+---------+----------+
  King   Queen    Rook    Bishop  Knight   Pawn
```

### 5.2 Template Method — within Piece hierarchy

**Where:** `Piece` provides the common constructor/getters; concrete subclasses fill in `isValidMove`.

**Why:** Avoids duplicating `color`, `type`, `toString()` in every subclass.

### 5.3 Null Object / optional capture — in Move

The `pieceCaptured` field is `null` when there is no capture. This is a lightweight alternative to an Optional or a separate CaptureMove subclass, which is pragmatic for a 90-minute interview.

### 5.4 Simulate-and-Revert instead of deep copy

Instead of building an entirely separate board for each legal-move check, we:
1. Apply the move to the real board (`applyMove`).
2. Test for check.
3. Revert (`undoMove`).

This is O(1) space per check rather than O(64) for a deep copy, and is correct as long as `isInCheck` only reads the board (it does).

---

## 6. Walkthrough of the Approach

### Step 1 — Entities first (5 min)
Identify the nouns: Board, Cell, Piece, Player, Move, Game. Quickly sketch the ownership: Game owns Board and Players; Board owns a 2D array of Cells; each Cell optionally holds a Piece.

### Step 2 — Piece movement (20 min)
Write the abstract `Piece` class and all six subclasses. Start with the simplest (Knight — no path check) and end with the most complex (Pawn — direction, double step, diagonal capture). For sliding pieces (Queen, Rook, Bishop), implement `Board.isPathClear()` once and delegate to it.

### Step 3 — Board setup (10 min)
Implement `Board` with `initializePieces()`. Lay out the standard starting position. Implement `applyMove` and `undoMove` as simple cell mutations.

### Step 4 — Check detection (15 min)
`isInCheck(color)`: find the King; iterate every enemy piece; call `piece.isValidMove(board, attackerCell, kingCell)`. This reuses the move-validation logic already written.

### Step 5 — Legal move filter (10 min)
`moveLeavesKingInCheck(from, to, color)`: apply move, call `isInCheck`, revert. `hasAnyLegalMove(color)`: nested loop over all pieces × all 64 squares, filter by `isValidMove` and `!moveLeavesKingInCheck`.

### Step 6 — Game class and status (10 min)
`makeMove` validates ownership, calls piece's `isValidMove`, calls `moveLeavesKingInCheck`, applies the move, then calls `updateStatus(opponent)` which uses `isInCheck` and `hasAnyLegalMove` to decide ACTIVE / WHITE_WINS / BLACK_WINS / STALEMATE.

### Step 7 — Main / console loop (5 min)
A simple `while (!game.isOver())` loop reads coordinates and calls `makeMove`.

### Complexity
- `isInCheck`: O(16 × 1) = O(1) amortised (at most 16 enemy pieces, each `isValidMove` is O(8) for sliding pieces).
- `hasAnyLegalMove`: O(16 pieces × 64 cells × isInCheck per candidate) = O(16 × 64 × 16) = O(16 384) — perfectly fast.

---

## 7. Correctness Notes

### 7.1 Why `isInCheck` reuses `isValidMove`

When we ask "can the enemy Rook at e1 attack the King at e8?" we call `rook.isValidMove(board, e1Cell, e8Cell)`. This correctly checks that the path is clear and that the destination is not occupied by a friendly piece (the King is the opposite colour, so it will not be blocked by that check). This double-use of the same logic means check detection cannot diverge from actual movement rules.

### 7.2 The simulate-and-revert contract

`moveLeavesKingInCheck` is only correct if nothing else modifies the board between `applyMove` and `undoMove`. The method is private and the call site is the only place these two calls appear together, so the invariant is easy to maintain.

### 7.3 Pawn direction convention

Row 0 is White's back rank (rank 1). White pawns on row 1 advance toward row 7 (`direction = +1`). Black pawns on row 6 advance toward row 0 (`direction = -1`). This must be consistent with `Board.initializePieces()`.

---

## 8. Extension: Timed Games and Online Multiplayer

### 8.1 Timed Games — Interface Design

```java
// Clock.java
public interface Clock {
    /** Start or resume the clock for the given player. */
    void start(Player player);

    /** Pause the clock for the given player. */
    void pause(Player player);

    /** Returns remaining time in milliseconds for the given player. */
    long getRemainingTimeMs(Player player);

    /** Returns true if the given player has flagged (time expired). */
    boolean hasExpired(Player player);

    /** Register a callback to be invoked when a player's time expires. */
    void onExpiry(Player player, Runnable callback);
}
```

```java
// ClockType.java (factory hint)
public enum ClockType {
    BLITZ,        //  e.g. 3+2 (3 min + 2 sec increment)
    RAPID,        //  e.g. 10+0
    CLASSICAL,    //  e.g. 90 min + 30 sec increment
    CORRESPONDENCE
}
```

**Integration:** `Game.makeMove()` calls `clock.pause(currentPlayer)` after a successful move and `clock.start(opponent)` before switching turns. A background thread (or a scheduled executor) polls `clock.hasExpired()` and calls `handleExpiry(Player)` which sets `GameStatus` to the appropriate winner.

```java
// In Game (addition only — no change to existing methods)
private Clock clock;  // injected via constructor overload

private void handleExpiry(Player expired) {
    status = (expired.getColor() == PieceColor.WHITE)
           ? GameStatus.BLACK_WINS
           : GameStatus.WHITE_WINS;
    System.out.println(expired.getName() + " ran out of time!");
}
```

### 8.2 Online Multiplayer — Interface Design

The key insight is to separate **move submission** from **move application**. In a local game both happen synchronously in `makeMove`. In a networked game:

1. A player submits a move to a server.
2. The server validates and applies it.
3. Both clients receive the updated game state.

```java
// MoveSubmitter.java  — called by the local client
public interface MoveSubmitter {
    /**
     * Submits a move to the remote game server.
     * Returns immediately; result is delivered via GameEventListener.
     */
    void submitMove(String gameId, int fromRow, int fromCol, int toRow, int toCol);
}
```

```java
// GameEventListener.java  — implemented by the local UI / game loop
public interface GameEventListener {
    /** Called when a move is accepted and applied by the server. */
    void onMoveMade(Move move, GameStatus newStatus);

    /** Called when a move is rejected (illegal). */
    void onMoveRejected(String reason);

    /** Called when the opponent disconnects. */
    void onOpponentDisconnected();

    /** Called when the game ends. */
    void onGameOver(GameStatus status);
}
```

```java
// GameServer.java  — server-side contract
public interface GameServer {
    /** Creates a new game; returns a unique game ID. */
    String createGame(Player white, Player black);

    /**
     * Validates and applies a move; broadcasts the result to both players
     * via their registered GameEventListeners.
     */
    MoveResult applyMove(String gameId, Player player,
                         int fromRow, int fromCol, int toRow, int toCol);

    /** Returns the current game state (useful for reconnection). */
    GameSnapshot getSnapshot(String gameId);
}
```

```java
// GameSnapshot.java  — serializable state for reconnection / spectators
public class GameSnapshot {
    private final String gameId;
    private final String[][] boardState;   // piece symbols or FEN
    private final PieceColor currentTurn;
    private final GameStatus status;
    private final List<String> moveHistory; // in algebraic notation

    // ... constructor, getters
}
```

**Architectural note:** The `Game` class itself does not change. The `GameServer` implementation wraps a `Game` instance per game session. Move validation remains in the same piece classes — there is no duplication of rules between client and server. The client can do optimistic validation (show illegal-move feedback immediately) while the server is the authoritative source.

---

## 9. What to Mention as Extensions (but not implement)

| Feature | Approach |
|---------|----------|
| **Castling** | Add `hasMoved` flag to `King` and `Rook`. In `King.isValidMove`, allow a 2-square horizontal move when both pieces have not moved, no pieces between them, King not in check, King does not pass through check. Rook teleports to the other side of the King. |
| **En passant** | Store the last double-step pawn move in `Game`. In `Pawn.isValidMove`, check if the target square is the en-passant square and the last move was a double-step by an adjacent pawn. |
| **Pawn promotion** | When a pawn reaches rank 8 (White) or rank 1 (Black), prompt the player to choose Queen, Rook, Bishop, or Knight. Replace the pawn piece on the board. |
| **Draw by 50-move rule** | Track a half-move clock in `Game`; reset on pawn move or capture. |
| **Draw by repetition** | Store board hashes in a map; if the same position occurs three times, offer a draw. |
| **Algebraic notation input** | Add a parser that converts "e2e4" or "Nf3" to row/col pairs before calling `makeMove`. |
