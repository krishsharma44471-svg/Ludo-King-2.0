import pygame
import random

# पायगेम सेटअप
pygame.init()

# स्क्रीन सेटिंग्स
screen_width, screen_height = 1080, 1920 # मोबाइल स्क्रीन के लिए बेहतर रेश्यो
screen = pygame.display.set_mode((screen_width, screen_height))
pygame.display.set_caption("Ludo Pro - Ultimate King Edition")

# कलर्स (Colors)
WHITE, BLACK, GRAY = (255, 255, 255), (0, 0, 0), (180, 180, 180)
GREEN, RED, YELLOW, BLUE = (34, 177, 76), (237, 28, 36), (255, 242, 0), (63, 72, 204)
DARK_BG = (15, 15, 15)

# बोर्ड कॉन्फ़िगरेशन
board_size = 1000
cell = board_size // 15
start_x = (screen_width - board_size) // 2
start_y = 300

# गेम स्टेट्स
INTRO, SELECT_PLAYER, PLAYING, WINNER = 0, 1, 2, 3
game_state = INTRO

# पाथ डेटा (Path Data)
master_path = [(6,13),(6,12),(6,11),(6,10),(6,9),(5,8),(4,8),(3,8),(2,8),(1,8),(0,8),(0,7),(0,6),(1,6),(2,6),(3,6),(4,6),(5,6),(6,5),(6,4),(6,3),(6,2),(6,1),(6,0),(7,0),(8,0),(8,1),(8,2),(8,3),(8,4),(8,5),(9,6),(10,6),(11,6),(12,6),(13,6),(14,6),(14,7),(14,8),(13,8),(12,8),(11,8),(10,8),(9,8),(8,9),(8,10),(8,11),(8,12),(8,13),(8,14),(7,14),(6,14)]
home_lanes = [[(1,7),(2,7),(3,7),(4,7),(5,7),(6,7)], [(7,1),(7,2),(7,3),(7,4),(7,5),(7,6)], [(13,7),(12,7),(11,7),(10,7),(9,7),(8,7)], [(7,13),(7,12),(7,11),(7,10),(7,9),(7,8)]]
start_indices = [13, 26, 39, 0] 
safe_points = [(1,6), (2,8), (6,13), (8,12), (13,8), (12,6), (8,1), (6,2)]
player_colors = [RED, YELLOW, BLUE, GREEN]
player_names = ["RED", "YELLOW", "BLUE", "GREEN"]

# वेरिएबल्स
dice_value, current_turn, dice_rolled = 1, 0, False
active_players, tokens = [], [[0,0,0,0] for _ in range(4)]
winners_list = []
turn_start_time = 0
TIMER_LIMIT = 30000 

dice_rect = pygame.Rect(screen_width//2 - 100, start_y + board_size + 100, 200, 150)
start_btn = pygame.Rect(screen_width//2 - 250, 800, 500, 150)

def draw_base_wings(x, y, color):
    pygame.draw.rect(screen, color, [x, y, cell*6, cell*6], 0, 15)
    pygame.draw.rect(screen, WHITE, [x+cell, y+cell, cell*4, cell*4], 0, 20)
    offsets = [(1.8, 1.8), (4.2, 1.8), (1.8, 4.2), (4.2, 4.2)]
    for ox, oy in offsets:
        pygame.draw.circle(screen, color, (x + ox*cell, y + oy*cell), cell//1.6, 2)
        pygame.draw.circle(screen, color, (x + ox*cell, y + oy*cell), cell//2.2)

def draw_star(cx, cy):
    size = 20
    pts = []
    for i in range(10):
        angle = i * (3.14159 / 5)
        r = size if i % 2 == 0 else size // 2
        pts.append((cx + r * pygame.math.Vector2(1, 0).rotate_rad(angle).x, 
                    cy + r * pygame.math.Vector2(1, 0).rotate_rad(angle).y))
    pygame.draw.polygon(screen, GRAY, pts)

def get_coord(p_idx, t_idx, pos):
    if pos == 0:
        base_x = start_x if p_idx in [0, 3] else start_x + 9*cell
        base_y = start_y if p_idx in [0, 1] else start_y + 9*cell
        offs = [(1.8, 1.8), (4.2, 1.8), (1.8, 4.2), (4.2, 4.2)]
        return base_x + offs[t_idx][0]*cell, base_y + offs[t_idx][1]*cell
    if 1 <= pos <= 51:
        gx, gy = master_path[(pos - 1 + start_indices[p_idx]) % 52]
        return start_x + gx*cell + cell//2, start_y + gy*cell + cell//2
    if 52 <= pos <= 57:
        gx, gy = home_lanes[p_idx][pos - 52]
        return start_x + gx*cell + cell//2, start_y + gy*cell + cell//2
    return start_x + 7.5*cell, start_y + 7.5*cell

def check_and_kill(p_idx, t_idx):
    pos = tokens[p_idx][t_idx]
    if pos == 0 or pos > 51: return False
    curr_global_idx = (pos - 1 + start_indices[p_idx]) % 52
    if master_path[curr_global_idx] in safe_points: return False
    killed = False
    for other_p in active_players:
        if other_p == p_idx: continue
        for other_t in range(4):
            if 1 <= tokens[other_p][other_t] <= 51:
                if (tokens[other_p][other_t] - 1 + start_indices[other_p]) % 52 == curr_global_idx:
                    tokens[other_p][other_t] = 0
                    killed = True
    return killed

def draw_board():
    screen.fill(DARK_BG)
    pygame.draw.rect(screen, WHITE, [start_x, start_y, board_size, board_size])
    for i in range(1, 6):
        pygame.draw.rect(screen, RED, [start_x + i*cell, start_y + 7*cell, cell, cell])
        pygame.draw.rect(screen, YELLOW, [start_x + 7*cell, start_y + i*cell, cell, cell])
        pygame.draw.rect(screen, BLUE, [start_x + (14-i)*cell, start_y + 7*cell, cell, cell])
        pygame.draw.rect(screen, GREEN, [start_x + 7*cell, start_y + (14-i)*cell, cell, cell])
    
    center = (start_x + 7.5*cell, start_y + 7.5*cell)
    pygame.draw.polygon(screen, RED, [(start_x+6*cell, start_y+6*cell), (start_x+6*cell, start_y+9*cell), center])
    pygame.draw.polygon(screen, YELLOW, [(start_x+6*cell, start_y+6*cell), (start_x+9*cell, start_y+6*cell), center])
    pygame.draw.polygon(screen, BLUE, [(start_x+9*cell, start_y+6*cell), (start_x+9*cell, start_y+9*cell), center])
    pygame.draw.polygon(screen, GREEN, [(start_x+6*cell, start_y+9*cell), (start_x+9*cell, start_y+9*cell), center])
    
    draw_base_wings(start_x, start_y, RED)
    draw_base_wings(start_x + 9*cell, start_y, YELLOW)
    draw_base_wings(start_x + 9*cell, start_y + 9*cell, BLUE)
    draw_base_wings(start_x, start_y + 9*cell, GREEN)
    
    for i in range(16):
        pygame.draw.line(screen, BLACK, (start_x+i*cell, start_y), (start_x+i*cell, start_y+board_size), 2)
        pygame.draw.line(screen, BLACK, (start_x, start_y+i*cell), (start_x+board_size, start_y+i*cell), 2)
    
    for (sx, sy) in safe_points:
        draw_star(start_x + sx*cell + cell//2, start_y + sy*cell + cell//2)
    
    for p in range(4):
        for t in range(4):
            if tokens[p][t] <= 57:
                tx, ty = get_coord(p, t, tokens[p][t])
                pygame.draw.circle(screen, BLACK, (tx, ty), cell//2 - 2)
                pygame.draw.circle(screen, player_colors[p], (tx, ty), cell//2 - 6)
                pygame.draw.circle(screen, WHITE, (tx-4, ty-4), 4)

# गेम लूप
running = True
clock = pygame.time.Clock()

while running:
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False
        
        if event.type == pygame.MOUSEBUTTONDOWN:
            if game_state == INTRO and start_btn.collidepoint(event.pos):
                game_state = SELECT_PLAYER
            elif game_state == SELECT_PLAYER:
                for i, p_count in enumerate([2, 3, 4]):
                    r = pygame.Rect(screen_width//2 - 200, 500 + i*200, 400, 120)
                    if r.collidepoint(event.pos):
                        active_players = [0, 2] if p_count == 2 else ([0, 1, 2] if p_count == 3 else [0, 1, 2, 3])
                        tokens = [[0,0,0,0] for _ in range(4)]
                        winners_list = []
                        current_turn = active_players[0]
                        turn_start_time = pygame.time.get_ticks()
                        game_state = PLAYING
            
            elif game_state == PLAYING:
                if dice_rect.collidepoint(event.pos) and not dice_rolled:
                    dice_value = random.randint(1, 6)
                    dice_rolled = True
                    possible_moves = [t for t in range(4) if (tokens[current_turn][t] == 0 and dice_value == 6) or (0 < tokens[current_turn][t] <= 57 - dice_value)]
                    if not possible_moves:
                        pygame.display.flip()
                        pygame.time.delay(800)
                        dice_rolled = False
                        current_turn = active_players[(active_players.index(current_turn)+1)%len(active_players)]
                        turn_start_time = pygame.time.get_ticks()

                elif dice_rolled:
                    for t in range(4):
                        tx, ty = get_coord(current_turn, t, tokens[current_turn][t])
                        if pygame.Rect(tx-50, ty-50, 100, 100).collidepoint(event.pos):
                            moved = False
                            in_goal = False
                            
                            if tokens[current_turn][t] == 0 and dice_value == 6:
                                tokens[current_turn][t] = 1
                                moved = True
                            elif tokens[current_turn][t] > 0 and tokens[current_turn][t] + dice_value <= 57:
                                tokens[current_turn][t] += dice_value
                                moved = True
                                if tokens[current_turn][t] == 57: in_goal = True
                            
                            if moved:
                                killed = check_and_kill(current_turn, t)
                                dice_rolled = False
                                if all(tk == 57 for tk in tokens[current_turn]):
                                    winners_list.append(player_names[current_turn])
                                    active_players.remove(current_turn)
                                    if len(active_players) <= 1:
                                        if active_players: winners_list.append(player_names[active_players[0]])
                                        game_state = WINNER
                                    else:
                                        current_turn = active_players[0]
                                else:
                                    if not (dice_value == 6 or killed or in_goal):
                                        current_turn = active_players[(active_players.index(current_turn)+1)%len(active_players)]
                                turn_start_time = pygame.time.get_ticks()
                                break

    # रेंडरिंग (UI)
    if game_state == INTRO:
        screen.fill(DARK_BG)
        font = pygame.font.SysFont(None, 150)
        title = font.render("LUDO KING", True, YELLOW)
        screen.blit(title, (screen_width//2 - title.get_width()//2, 400))
        pygame.draw.rect(screen, GREEN, start_btn, 0, 30)
        btn_txt = pygame.font.SysFont(None, 80).render("START", True, WHITE)
        screen.blit(btn_txt, (start_btn.centerx-btn_txt.get_width()//2, start_btn.centery-btn_txt.get_height()//2))
    
    elif game_state == SELECT_PLAYER:
        screen.fill(DARK_BG)
        for i, txt in enumerate(["2 PLAYERS", "3 PLAYERS", "4 PLAYERS"]):
            r = pygame.Rect(screen_width//2 - 200, 500 + i*200, 400, 120)
            pygame.draw.rect(screen, player_colors[i], r, 0, 20)
            t_rnd = pygame.font.SysFont(None, 60).render(txt, True, WHITE)
            screen.blit(t_rnd, (r.centerx-t_rnd.get_width()//2, r.centery-t_rnd.get_height()//2))
    
    elif game_state == PLAYING:
        draw_board()
        now = pygame.time.get_ticks()
        elapsed = now - turn_start_time
        remaining = max(0, (TIMER_LIMIT - elapsed) // 1000)
        
        if elapsed >= TIMER_LIMIT:
            dice_rolled = False
            current_turn = active_players[(active_players.index(current_turn)+1)%len(active_players)]
            turn_start_time = pygame.time.get_ticks()

        time_txt = pygame.font.SysFont(None, 60).render(f"TURN: {player_names[current_turn]} | {remaining}s", True, WHITE)
        screen.blit(time_txt, (screen_width//2 - time_txt.get_width()//2, start_y - 100))

        pygame.draw.rect(screen, player_colors[current_turn], dice_rect, 0, 20)
        pygame.draw.rect(screen, WHITE, dice_rect, 5, 20)
        d_val = pygame.font.SysFont(None, 120).render(str(dice_value), True, WHITE)
        screen.blit(d_val, d_val.get_rect(center=dice_rect.center))
        
    elif game_state == WINNER:
        screen.fill(DARK_BG)
        for i, name in enumerate(winners_list):
            w_surf = pygame.font.SysFont(None, 100).render(f"{i+1}. {name}", True, WHITE)
            screen.blit(w_surf, (screen_width//2 - w_surf.get_width()//2, 400 + i*150))

    pygame.display.flip()
    clock.tick(30)

pygame.quit()
