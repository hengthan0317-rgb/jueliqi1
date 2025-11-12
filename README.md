import pygame
import sys
import math
from enum import Enum
from typing import List, Tuple, Dict, Set, Optional

# 初始化pygame
pygame.init()

# 常量定义
SCREEN_WIDTH = 900
SCREEN_HEIGHT = 750
BOARD_SIZE = 8
CELL_SIZE = 60
MARGIN = 50
PIECE_RADIUS = 25

# 颜色定义
WHITE = (255, 255, 255)
BLACK = (0, 0, 0)
LIGHT_COLOR = (240, 240, 240)
DARK_COLOR = (200, 220, 240)
RED = (231, 76, 60)      # A
GREEN = (46, 204, 113)   # B
BLUE = (52, 152, 219)    # C
YELLOW = (241, 196, 15)  # D
HINT_COLOR = (76, 175, 80)
SWAP_COLOR = (233, 30, 99)
TELEPORT_COLOR = (33, 150, 243)
SELECT_COLOR = (255, 179, 0)
GOAL_COLOR = (100, 100, 100, 128)
PANEL_COLOR = (250, 250, 250)
PANEL_BORDER = (200, 200, 200)

class GameMode(Enum):
    TWO_PLAYER = 2
    FOUR_PLAYER = 4

class Piece:
    def __init__(self, player: str, row: int, col: int, piece_id: int):
        self.player = player
        self.row = row
        self.col = col
        self.id = piece_id
        self.selected = False
    
    def get_color(self):
        colors = {
            'A': RED,
            'B': GREEN, 
            'C': BLUE,
            'D': YELLOW
        }
        return colors.get(self.player, WHITE)

class GameState:
    def __init__(self):
        self.mode = GameMode.FOUR_PLAYER
        self.players = ['A', 'B', 'C', 'D']
        self.current_player_index = 0
        self.pieces: Dict[str, List[Piece]] = {}
        self.goals: Dict[str, Set[Tuple[int, int]]] = {}
        self.selected_piece: Optional[Piece] = None
        self.valid_moves: List[Tuple[int, int]] = []
        self.swap_moves: List[Tuple[int, int]] = []
        self.teleport_moves: List[Tuple[int, int]] = []
        self.ranks: List[str] = []
        self.finished_players: Set[str] = set()
        self.show_hints = True
        self.next_piece_id = 1
    
    def get_current_player(self) -> str:
        return self.players[self.current_player_index]
    
    def next_turn(self):
        while True:
            self.current_player_index = (self.current_player_index + 1) % len(self.players)
            if self.players[self.current_player_index] not in self.finished_players:
                break
    
    def is_game_over(self) -> bool:
        return len(self.finished_players) >= len(self.players) - 1

class CornerForceGame:
    def __init__(self):
        self.screen = pygame.display.set_mode((SCREEN_WIDTH, SCREEN_HEIGHT))
        pygame.display.set_caption("角力棋 8×8（Python版）")
        self.clock = pygame.time.Clock()
        self.font = pygame.font.SysFont('microsoftyahei', 24)
        self.small_font = pygame.font.SysFont('microsoftyahei', 18)
        self.state = GameState()
        self.initialize_game()
    
    def initialize_game(self):
        """初始化游戏状态"""
        self.state = GameState()
        
        # 设置玩家顺序
        if self.state.mode == GameMode.TWO_PLAYER:
            self.state.players = ['A', 'B']
        
        # 初始化棋子位置
        self.initialize_pieces()
        
        # 初始化目标区域
        self.initialize_goals()
    
    def initialize_pieces(self):
        """初始化棋子位置"""
        self.state.pieces = {}
        
        corners = {
            'A': [(0, 0), (0, 1), (0, 2), (1, 0), (1, 1), (1, 2), (2, 0), (2, 1), (2, 2)],  # 左上角
            'B': [(0, 5), (0, 6), (0, 7), (1, 5), (1, 6), (1, 7), (2, 5), (2, 6), (2, 7)],  # 右上角
            'C': [(5, 5), (5, 6), (5, 7), (6, 5), (6, 6), (6, 7), (7, 5), (7, 6), (7, 7)],  # 右下角
            'D': [(5, 0), (5, 1), (5, 2), (6, 0), (6, 1), (6, 2), (7, 0), (7, 1), (7, 2)]   # 左下角
        }
        
        if self.state.mode == GameMode.TWO_PLAYER:
            corners = {
                'A': corners['A'],  # 左上角
                'B': corners['C']   # 右下角
            }
        
        for player in self.state.players:
            self.state.pieces[player] = []
            for row, col in corners[player]:
                piece = Piece(player, row, col, self.state.next_piece_id)
                self.state.pieces[player].append(piece)
                self.state.next_piece_id += 1
    
    def initialize_goals(self):
        """初始化目标区域"""
        self.state.goals = {}
        
        goal_corners = {
            'A': [(5, 5), (5, 6), (5, 7), (6, 5), (6, 6), (6, 7), (7, 5), (7, 6), (7, 7)],  # 右下角
            'B': [(5, 0), (5, 1), (5, 2), (6, 0), (6, 1), (6, 2), (7, 0), (7, 1), (7, 2)],  # 左下角
            'C': [(0, 0), (0, 1), (0, 2), (1, 0), (1, 1), (1, 2), (2, 0), (2, 1), (2, 2)],  # 左上角
            'D': [(0, 5), (0, 6), (0, 7), (1, 5), (1, 6), (1, 7), (2, 5), (2, 6), (2, 7)]   # 右上角
        }
        
        if self.state.mode == GameMode.TWO_PLAYER:
            goal_corners = {
                'A': goal_corners['A'],  # 右下角
                'B': goal_corners['C']   # 左上角
            }
        
        for player in self.state.players:
            self.state.goals[player] = set(goal_corners[player])
    
    def get_piece_at(self, row: int, col: int) -> Optional[Piece]:
        """获取指定位置的棋子"""
        for player_pieces in self.state.pieces.values():
            for piece in player_pieces:
                if piece.row == row and piece.col == col:
                    return piece
        return None
    
    def is_valid_position(self, row: int, col: int) -> bool:
        """检查位置是否在棋盘内"""
        return 0 <= row < BOARD_SIZE and 0 <= col < BOARD_SIZE
    
    def calculate_valid_moves(self, piece: Piece):
        """计算有效移动位置"""
        self.state.valid_moves = []
        self.state.swap_moves = []
        self.state.teleport_moves = []
        
        current_player = piece.player
        
        # 基本移动（上下左右）
        directions = [(0, 1), (0, -1), (1, 0), (-1, 0)]
        for dr, dc in directions:
            new_row, new_col = piece.row + dr, piece.col + dc
            if self.is_valid_position(new_row, new_col):
                target_piece = self.get_piece_at(new_row, new_col)
                if target_piece is None:
                    # 空位置，可以移动
                    self.state.valid_moves.append((new_row, new_col))
                elif target_piece.player != current_player:
                    # 敌方棋子，检查是否可以强制换位
                    if self.can_force_swap(piece, target_piece):
                        self.state.swap_moves.append((new_row, new_col))
        
        # 传送机制（简化版：对角线对称传送）
        teleport_pos = (piece.col, piece.row)  # [r,c]→[c,r]
        if (self.is_valid_position(teleport_pos[0], teleport_pos[1]) and 
            self.get_piece_at(teleport_pos[0], teleport_pos[1]) is None):
            self.state.teleport_moves.append(teleport_pos)
    
    def can_force_swap(self, moving_piece: Piece, target_piece: Piece) -> bool:
        """检查是否可以强制换位"""
        # 简化版：需要至少2个友方棋子相邻
        current_player = moving_piece.player
        friendly_count = 0
        
        directions = [(0, 1), (0, -1), (1, 0), (-1, 0)]
        for dr, dc in directions:
            check_row, check_col = target_piece.row + dr, target_piece.col + dc
            if self.is_valid_position(check_row, check_col):
                adjacent_piece = self.get_piece_at(check_row, check_col)
                if adjacent_piece and adjacent_piece.player == current_player:
                    friendly_count += 1
        
        return friendly_count >= 2
    
    def move_piece(self, piece: Piece, target_row: int, target_col: int):
        """移动棋子"""
        target_piece = self.get_piece_at(target_row, target_col)
        
        if (target_row, target_col) in self.state.valid_moves:
            # 普通移动
            piece.row, piece.col = target_row, target_col
            
        elif (target_row, target_col) in self.state.swap_moves and target_piece:
            # 强制换位
            piece.row, piece.col, target_piece.row, target_piece.col = (
                target_piece.row, target_piece.col, piece.row, piece.col
            )
            
        elif (target_row, target_col) in self.state.teleport_moves:
            # 传送移动
            piece.row, piece.col = target_row, target_col
        
        # 检查是否完成目标
        self.check_goal_completion(piece.player)
        
        # 切换到下一玩家
        self.state.next_turn()
        self.state.selected_piece = None
    
    def check_goal_completion(self, player: str):
        """检查玩家是否完成目标"""
        if player in self.state.finished_players:
            return
        
        # 检查所有棋子是否都在目标区域
        all_in_goal = all(
            (piece.row, piece.col) in self.state.goals[player]
            for piece in self.state.pieces[player]
        )
        
        if all_in_goal and player not in self.state.finished_players:
            self.state.finished_players.add(player)
            self.state.ranks.append(player)
    
    def draw_board(self):
        """绘制棋盘"""
        board_left = MARGIN
        board_top = 100  # 为控制面板留出空间
        
        # 绘制棋盘格子
        for row in range(BOARD_SIZE):
            for col in range(BOARD_SIZE):
                x = board_left + col * CELL_SIZE
                y = board_top + row * CELL_SIZE
                
                # 格子颜色
                color = LIGHT_COLOR if (row + col) % 2 == 0 else DARK_COLOR
                pygame.draw.rect(self.screen, color, (x, y, CELL_SIZE, CELL_SIZE))
                pygame.draw.rect(self.screen, BLACK, (x, y, CELL_SIZE, CELL_SIZE), 1)
                
                # 绘制目标区域
                for player, goals in self.state.goals.items():
                    if (row, col) in goals:
                        s = pygame.Surface((CELL_SIZE, CELL_SIZE), pygame.SRCALPHA)
                        s.fill((*GOAL_COLOR[:3], 100))
                        self.screen.blit(s, (x, y))
                        pygame.draw.rect(self.screen, BLACK, (x, y, CELL_SIZE, CELL_SIZE), 2)
    
    def draw_pieces(self):
        """绘制棋子"""
        board_left = MARGIN
        board_top = 100
        
        for player_pieces in self.state.pieces.values():
            for piece in player_pieces:
                x = board_left + piece.col * CELL_SIZE + CELL_SIZE // 2
                y = board_top + piece.row * CELL_SIZE + CELL_SIZE // 2
                
                # 绘制棋子
                color = piece.get_color()
                pygame.draw.circle(self.screen, color, (x, y), PIECE_RADIUS)
                pygame.draw.circle(self.screen, BLACK, (x, y), PIECE_RADIUS, 2)
                
                # 绘制棋子标识
                text = self.small_font.render(piece.player, True, WHITE if piece.player != 'D' else BLACK)
                text_rect = text.get_rect(center=(x, y))
                self.screen.blit(text, text_rect)
                
                # 绘制选中效果
                if piece.selected:
                    pygame.draw.circle(self.screen, SELECT_COLOR, (x, y), PIECE_RADIUS + 3, 3)
    
    def draw_hints(self):
        """绘制移动提示"""
        if not self.state.show_hints or not self.state.selected_piece:
            return
        
        board_left = MARGIN
        board_top = 100
        
        # 绘制有效移动位置
        for row, col in self.state.valid_moves:
            x = board_left + col * CELL_SIZE + CELL_SIZE // 2
            y = board_top + row * CELL_SIZE + CELL_SIZE // 2
            pygame.draw.circle(self.screen, HINT_COLOR, (x, y), 15, 3)
        
        # 绘制强制换位位置
        for row, col in self.state.swap_moves:
            x = board_left + col * CELL_SIZE + CELL_SIZE // 2
            y = board_top + row * CELL_SIZE + CELL_SIZE // 2
            pygame.draw.circle(self.screen, SWAP_COLOR, (x, y), 15, 3)
        
        # 绘制传送位置
        for row, col in self.state.teleport_moves:
            x = board_left + col * CELL_SIZE + CELL_SIZE // 2
            y = board_top + row * CELL_SIZE + CELL_SIZE // 2
            pygame.draw.circle(self.screen, TELEPORT_COLOR, (x, y), 15, 3)
    
    def draw_ui(self):
        """绘制用户界面"""
        # 绘制标题
        title = self.font.render("角力棋 8×8（Python版）", True, BLACK)
        self.screen.blit(title, (MARGIN, 20))
        
        # 绘制控制面板
        panel_rect = pygame.Rect(MARGIN, 50, SCREEN_WIDTH - 2 * MARGIN, 40)
        pygame.draw.rect(self.screen, PANEL_COLOR, panel_rect)
        pygame.draw.rect(self.screen, PANEL_BORDER, panel_rect, 2)
        
        # 绘制当前玩家
        current_player_text = f"当前玩家: {self.state.get_current_player()}"
        player_text = self.small_font.render(current_player_text, True, BLACK)
        self.screen.blit(player_text, (MARGIN + 10, 60))
        
        # 绘制名次
        ranks_text = f"名次: {' > '.join(self.state.ranks) if self.state.ranks else '—'}"
        ranks_render = self.small_font.render(ranks_text, True, BLACK)
        self.screen.blit(ranks_render, (MARGIN + 200, 60))
        
        # 绘制游戏模式
        mode_text = f"模式: {'二人' if self.state.mode == GameMode.TWO_PLAYER else '四人'}模式"
        mode_render = self.small_font.render(mode_text, True, BLACK)
        self.screen.blit(mode_render, (MARGIN + 400, 60))
        
        # 绘制提示说明
        legend_y = SCREEN_HEIGHT - 80
        legends = [
            ("合法落点", HINT_COLOR),
            ("强制换位", SWAP_COLOR),
            ("传送位置", TELEPORT_COLOR)
        ]
        
        for i, (text, color) in enumerate(legends):
            x = MARGIN + i * 150
            pygame.draw.circle(self.screen, color, (x, legend_y), 8)
            legend_text = self.small_font.render(text, True, BLACK)
            self.screen.blit(legend_text, (x + 15, legend_y - 8))
        
        # 绘制游戏说明
        instructions = [
            "游戏规则:",
            "1. 将己方所有棋子移动到对角目标区域",
            "2. 点击棋子选择，点击目标位置移动",
            "3. 绿色:可移动位置，粉色:可强制换位，蓝色:传送位置",
            "4. 按R重置游戏，按H切换提示显示"
        ]
        
        for i, instruction in enumerate(instructions):
            inst_text = self.small_font.render(instruction, True, BLACK)
            self.screen.blit(inst_text, (MARGIN, SCREEN_HEIGHT - 150 + i * 20))
    
    def handle_click(self, pos: Tuple[int, int]):
        """处理鼠标点击"""
        board_left = MARGIN
        board_top = 100
        
        # 计算点击的棋盘位置
        col = (pos[0] - board_left) // CELL_SIZE
        row = (pos[1] - board_top) // CELL_SIZE
        
        if not (0 <= row < BOARD_SIZE and 0 <= col < BOARD_SIZE):
            return
        
        clicked_piece = self.get_piece_at(row, col)
        current_player = self.state.get_current_player()
        
        if clicked_piece and clicked_piece.player == current_player:
            # 点击了己方棋子，选择它
            if self.state.selected_piece:
                self.state.selected_piece.selected = False
            self.state.selected_piece = clicked_piece
            clicked_piece.selected = True
            self.calculate_valid_moves(clicked_piece)
            
        elif self.state.selected_piece:
            # 点击了空位置或其他玩家的棋子，尝试移动
            target_pos = (row, col)
            if (target_pos in self.state.valid_moves or 
                target_pos in self.state.swap_moves or 
                target_pos in self.state.teleport_moves):
                
                self.move_piece(self.state.selected_piece, row, col)
    
    def run(self):
        """运行游戏主循环"""
        running = True
        
        while running:
            for event in pygame.event.get():
                if event.type == pygame.QUIT:
                    running = False
                elif event.type == pygame.MOUSEBUTTONDOWN:
                    if event.button == 1:  # 左键点击
                        self.handle_click(event.pos)
                elif event.type == pygame.KEYDOWN:
                    if event.key == pygame.K_r:  # 按R重置游戏
                        self.initialize_game()
                    elif event.key == pygame.K_h:  # 按H切换提示
                        self.state.show_hints = not self.state.show_hints
                    elif event.key == pygame.K_m:  # 按M切换模式
                        if self.state.mode == GameMode.TWO_PLAYER:
                            self.state.mode = GameMode.FOUR_PLAYER
                        else:
                            self.state.mode = GameMode.TWO_PLAYER
                        self.initialize_game()
            
            # 绘制游戏
            self.screen.fill(WHITE)
            self.draw_board()
            self.draw_hints()
            self.draw_pieces()
            self.draw_ui()
            
            pygame.display.flip()
            self.clock.tick(60)
        
        pygame.quit()
        sys.exit()

if __name__ == "__main__":
    game = CornerForceGame()
    game.run()
