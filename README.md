# 11# -*- coding: utf-8 -*-
"""
中国象棋小游戏 - Python + Tkinter
单文件实现，运行: python3 xiangqi.py
"""

import tkinter as tk
from tkinter import messagebox

# ============================================================
# 常量与棋子定义
# ============================================================
ROWS, COLS = 10, 9
CELL = 60           # 每格像素
MARGIN = 30         # 棋盘外边距
CANVAS_W = MARGIN * 2 + (COLS - 1) * CELL   # 540
CANVAS_H = MARGIN * 2 + (ROWS - 1) * CELL   # 600

EMPTY = None

PIECE_NAMES = {
    'K': '帅', 'A': '仕', 'B': '相', 'N': '马', 'R': '车', 'C': '炮', 'P': '兵',
    'k': '将', 'a': '士', 'b': '象', 'n': '马', 'r': '车', 'c': '炮', 'p': '卒',
}

RED, BLACK = 'red', 'black'

def piece_color(p):
    if p is None:
        return None
    return RED if p.isupper() else BLACK

def piece_type(p):
    return p.upper() if p else None

def on_board(r, c):
    return 0 <= r < ROWS and 0 <= c < COLS

def in_palace(r, c, color):
    if not (3 <= c <= 5):
        return False
    if color == RED:
        return 7 <= r <= 9
    else:
        return 0 <= r <= 2

def initial_board():
    board = [[EMPTY] * COLS for _ in range(ROWS)]
    back_black = ['r', 'n', 'b', 'a', 'k', 'a', 'b', 'n', 'r']
    for c, p in enumerate(back_black):
        board[0][c] = p
    board[2][1] = 'c'
    board[2][7] = 'c'
    for c in (0, 2, 4, 6, 8):
        board[3][c] = 'p'
    back_red = ['R', 'N', 'B', 'A', 'K', 'A', 'B', 'N', 'R']
    for c, p in enumerate(back_red):
        board[9][c] = p
    board[7][1] = 'C'
    board[7][7] = 'C'
    for c in (0, 2, 4, 6, 8):
        board[6][c] = 'P'
    return board

# ============================================================
# 走法生成（伪合法，不考虑将军过滤）
# ============================================================
def gen_moves(board, r, c):
    piece = board[r][c]
    if piece is None:
        return []
    color = piece_color(piece)
    ptype = piece_type(piece)
    moves = []

    if ptype == 'K':
        for dr, dc in [(-1, 0), (1, 0), (0, -1), (0, 1)]:
            nr, nc = r + dr, c + dc
            if on_board(nr, nc) and in_palace(nr, nc, color):
                if board[nr][nc] is None or piece_color(board[nr][nc]) != color:
                    moves.append((nr, nc))

    elif ptype == 'A':
        for dr, dc in [(-1, -1), (-1, 1), (1, -1), (1, 1)]:
            nr, nc = r + dr, c + dc
            if on_board(nr, nc) and in_palace(nr, nc, color):
                if board[nr][nc] is None or piece_color(board[nr][nc]) != color:
                    moves.append((nr, nc))

    elif ptype == 'B':
        for dr, dc in [(-2, -2), (-2, 2), (2, -2), (2, 2)]:
            nr, nc = r + dr, c + dc
            if not on_board(nr, nc):
                continue
            if board[r + dr // 2][c + dc // 2] is not None:
                continue
            if color == RED and nr < 5:
                continue
            if color == BLACK and nr > 4:
                continue
            if board[nr][nc] is None or piece_color(board[nr][nc]) != color:
                moves.append((nr, nc))

    elif ptype == 'N':
        candidates = [
            (-2, -1, -1, 0), (-2, 1, -1, 0),
            (2, -1, 1, 0), (2, 1, 1, 0),
            (-1, -2, 0, -1), (1, -2, 0, -1),
            (-1, 2, 0, 1), (1, 2, 0, 1),
        ]
        for dr, dc, br, bc in candidates:
            nr, nc = r + dr, c + dc
            if not on_board(nr, nc):
                continue
            if board[r + br][c + bc] is not None:
                continue
            if board[nr][nc] is None or piece_color(board[nr][nc]) != color:
                moves.append((nr, nc))

    elif ptype == 'R':
        for dr, dc in [(-1, 0), (1, 0), (0, -1), (0, 1)]:
            nr, nc = r + dr, c + dc
            while on_board(nr, nc):
                target = board[nr][nc]
                if target is None:
                    moves.append((nr, nc))
                else:
                    if piece_color(target) != color:
                        moves.append((nr, nc))
                    break
                nr += dr
                nc += dc

    elif ptype == 'C':
        for dr, dc in [(-1, 0), (1, 0), (0, -1), (0, 1)]:
            nr, nc = r + dr, c + dc
            jumped = False
            while on_board(nr, nc):
                target = board[nr][nc]
                if not jumped:
                    if target is None:
                        moves.append((nr, nc))
                    else:
                        jumped = True
                else:
                    if target is not None:
                        if piece_color(target) != color:
                            moves.append((nr, nc))
                        break
                nr += dr
                nc += dc

    elif ptype == 'P':
        forward = -1 if color == RED else 1
        nr, nc = r + forward, c
        if on_board(nr, nc):
            if board[nr][nc] is None or piece_color(board[nr][nc]) != color:
                moves.append((nr, nc))
        crossed = (color == RED and r >= 5) or (color == BLACK and r <= 4)
        if crossed:
            for dc in (-1, 1):
                nr2, nc2 = r, c + dc
                if on_board(nr2, nc2):
                    if board[nr2][nc2] is None or piece_color(board[nr2][nc2]) != color:
                        moves.append((nr2, nc2))

    return moves

def find_king(board, color):
    target = 'K' if color == RED else 'k'
    for r in range(ROWS):
        for c in range(COLS):
            if board[r][c] == target:
                return (r, c)
    return None

def kings_face_each_other(board):
    r1c1 = find_king(board, RED)
    r2c2 = find_king(board, BLACK)
    if r1c1 is None or r2c2 is None:
        return False
    r1, c1 = r1c1
    r2, c2 = r2c2
    if c1 != c2:
        return False
    lo, hi = sorted([r1, r2])
    for r in range(lo + 1, hi):
        if board[r][c1] is not None:
            return False
    return True

def is_in_check(board, color):
    if kings_face_each_other(board):
        return True
    kr, kc = find_king(board, color)
    if kr is None:
        return True
    opp = BLACK if color == RED else RED
    for r in range(ROWS):
        for c in range(COLS):
            p = board[r][c]
            if p is None or piece_color(p) != opp:
                continue
            if (kr, kc) in gen_moves(board, r, c):
                return True
    return False

def legal_moves(board, r, c):
    piece = board[r][c]
    if piece is None:
        return []
    color = piece_color(piece)
    result = []
    for nr, nc in gen_moves(board, r, c):
        captured = board[nr][nc]
        board[nr][nc] = piece
        board[r][c] = EMPTY
        if not is_in_check(board, color):
            result.append((nr, nc))
        board[r][c] = piece
        board[nr][nc] = captured
    return result

def has_any_legal_move(board, color):
    for r in range(ROWS):
        for c in range(COLS):
            p = board[r][c]
            if p is None or piece_color(p) != color:
                continue
            if legal_moves(board, r, c):
                return True
    return False

# ============================================================
# GUI
# ============================================================
class XiangqiGame:
    def __init__(self, root):
        self.root = root
        self.root.title('中国象棋')
        self.root.resizable(False, False)

        self.board = initial_board()
        self.current = RED
        self.selected = None
        self.legal_targets = []
        self.last_move = None
        self.game_over = False

        self.canvas = tk.Canvas(root, width=CANVAS_W, height=CANVAS_H,
                                bg='#F5DEB3')
        self.canvas.pack()
        self.canvas.bind('<Button-1>', self.on_click)

        self.status = tk.Label(root, text='红方先行', font=('Microsoft YaHei', 12))
        self.status.pack(side=tk.BOTTOM, pady=6)

        self.draw()

    def draw(self):
        self.canvas.delete('all')
        self.draw_board_lines()
        self.draw_pieces()
        self.draw_markers()

    def draw_board_lines(self):
        for r in range(ROWS):
            y = MARGIN + r * CELL
            self.canvas.create_line(MARGIN, y, MARGIN + (COLS - 1) * CELL, y)
        for c in range(COLS):
            x = MARGIN + c * CELL
            if c == 0 or c == COLS - 1:
                self.canvas.create_line(x, MARGIN, x, MARGIN + (ROWS - 1) * CELL)
            else:
                self.canvas.create_line(x, MARGIN, x, MARGIN + 4 * CELL)
                self.canvas.create_line(x, MARGIN + 5 * CELL, x, MARGIN + 9 * CELL)
        self.canvas.create_line(MARGIN + 3 * CELL, MARGIN,
                                MARGIN + 5 * CELL, MARGIN + 2 * CELL)
        self.canvas.create_line(MARGIN + 5 * CELL, MARGIN,
                                MARGIN + 3 * CELL, MARGIN + 2 * CELL)
        self.canvas.create_line(MARGIN + 3 * CELL, MARGIN + 7 * CELL,
                                MARGIN + 5 * CELL, MARGIN + 9 * CELL)
        self.canvas.create_line(MARGIN + 5 * CELL, MARGIN + 7 * CELL,
                                MARGIN + 3 * CELL, MARGIN + 9 * CELL)
        mid_y = MARGIN + 4.5 * CELL
        self.canvas.create_text(MARGIN + 1.5 * CELL, mid_y,
                                text='楚  河', font=('KaiTi', 22, 'bold'),
                                fill='#8B4513')
        self.canvas.create_text(MARGIN + 6.5 * CELL, mid_y,
                                text='汉  界', font=('KaiTi', 22, 'bold'),
                                fill='#8B4513')

    def draw_pieces(self):
        for r in range(ROWS):
            for c in range(COLS):
                p = self.board[r][c]
                if p is None:
                    continue
                x = MARGIN + c * CELL
                y = MARGIN + r * CELL
                color = piece_color(p)
                name = PIECE_NAMES[p]
                fill = '#FFF8DC'
                self.canvas.create_oval(x - 25, y - 25, x + 25, y + 25,
                                        fill=fill, outline='#333', width=2)
                self.canvas.create_oval(x - 22, y - 22, x + 22, y + 22,
                                        fill='',
                                        outline='#B8860B' if color == RED else '#2F4F4F',
                                        width=1)
                text_color = '#B22222' if color == RED else '#1C1C1C'
                self.canvas.create_text(x, y, text=name,
                                        font=('KaiTi', 22, 'bold'),
                                        fill=text_color)

    def draw_markers(self):
        if self.last_move:
            (sr, sc), (er, ec) = self.last_move
            for r, c in [(sr, sc), (er, ec)]:
                x = MARGIN + c * CELL
                y = MARGIN + r * CELL
                self.canvas.create_rectangle(x - 26, y - 26, x + 26, y + 26,
                                             outline='#1E90FF', width=2, dash=(4, 3))
        if self.selected:
            r, c = self.selected
            x = MARGIN + c * CELL
            y = MARGIN + r * CELL
            self.canvas.create_rectangle(x - 28, y - 28, x + 28, y + 28,
                                         outline='#FF4500', width=3)
        for nr, nc in self.legal_targets:
            x = MARGIN + nc * CELL
            y = MARGIN + nr * CELL
            target = self.board[nr][nc]
            if target is None:
                self.canvas.create_oval(x - 8, y - 8, x + 8, y + 8,
                                        fill='#32CD32', outline='')
            else:
                self.canvas.create_oval(x - 28, y - 28, x + 28, y + 28,
                                        outline='#FF6347', width=3)

    def pixel_to_cell(self, x, y):
        c = round((x - MARGIN) / CELL)
        r = round((y - MARGIN) / CELL)
        if on_board(r, c):
            return (r, c)
        return None

    def on_click(self, event):
        if self.game_over:
            return
        cell = self.pixel_to_cell(event.x, event.y)
        if cell is None:
            return
        r, c = cell
        clicked_piece = self.board[r][c]

        if self.selected is None:
            if clicked_piece and piece_color(clicked_piece) == self.current:
                self.selected = (r, c)
                self.legal_targets = legal_moves(self.board, r, c)
                self.draw()
        else:
            sr, sc = self.selected
            if (r, c) == self.selected:
                self.selected = None
                self.legal_targets = []
                self.draw()
                return
            if clicked_piece and piece_color(clicked_piece) == self.current:
                self.selected = (r, c)
                self.legal_targets = legal_moves(self.board, r, c)
                self.draw()
                return
            if (r, c) in self.legal_targets:
                self.make_move(sr, sc, r, c)

    def make_move(self, sr, sc, er, ec):
        piece = self.board[sr][sc]
        self.board[er][ec] = piece
        self.board[sr][sc] = EMPTY
        self.last_move = ((sr, sc), (er, ec))
        self.selected = None
        self.legal_targets = []

        self.current = BLACK if self.current == RED else RED
        opp_name = '黑方' if self.current == BLACK else '红方'

        if not has_any_legal_move(self.board, self.current):
            winner_name = '红方' if self.current == BLACK else '黑方'
            self.game_over = True
            self.draw()
            messagebox.showinfo('游戏结束', f'{winner_name} 获胜！')
            self.status.config(text=f'{winner_name} 获胜！点击"新游戏"重开')
            return

        if is_in_check(self.board, self.current):
            self.status.config(text=f'{opp_name}行动 —— 将军！')
        else:
            self.status.config(text=f'{opp_name}行动')

        self.draw()

def main():
    root = tk.Tk()
    game = XiangqiGame(root)
    btn = tk.Button(root, text='新游戏',
                    command=lambda: reset_game(game),
                    font=('Microsoft YaHei', 10))
    btn.pack(side=tk.RIGHT, padx=10, pady=4)
    root.mainloop()

def reset_game(game):
    game.board = initial_board()
    game.current = RED
    game.selected = None
    game.legal_targets = []
    game.last_move = None
    game.game_over = False
    game.status.config(text='红方先行')
    game.draw()

if __name__ == '__main__':
    main()
