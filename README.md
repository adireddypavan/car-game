# car-game
#pip install pygame
using python program designed a cargame 
import pygame
import random
import sys

# --- Config ---
WIDTH, HEIGHT = 480, 640
LANE_COUNT = 3
ROAD_MARGIN = 60
LANE_WIDTH = (WIDTH - 2 * ROAD_MARGIN) // LANE_COUNT
FPS = 60

PLAYER_WIDTH, PLAYER_HEIGHT = 40, 70
ENEMY_WIDTH, ENEMY_HEIGHT = 40, 70
PLAYER_SPEED = 6
ENEMY_MIN_SPEED = 4
ENEMY_MAX_SPEED = 9
SPAWN_INTERVAL_MS = 900

# Colors
BG = (25, 25, 25)
ROAD = (35, 35, 35)
LINE = (200, 200, 200)
TEXT = (240, 240, 240)

PLAYER_COLOR = (50, 200, 255)
ENEMY_COLOR = (255, 80, 80)

pygame.init()
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Lane Runner")
clock = pygame.time.Clock()
font = pygame.font.SysFont("consolas", 24)
big_font = pygame.font.SysFont("consolas", 42)

# --- Lane helpers ---
def lane_x(idx):
    return ROAD_MARGIN + idx * LANE_WIDTH + LANE_WIDTH // 2

def clamp(val, lo, hi):
    return max(lo, min(hi, val))

# --- Entities ---
class Car(pygame.sprite.Sprite):
    def __init__(self, x, y, w, h, color, speed=0):
        super().__init__()
        self.image = pygame.Surface((w, h), pygame.SRCALPHA)
        self.color = color
        pygame.draw.rect(self.image, color, (0, 0, w, h), border_radius=6)
        # simple windshield stripe
        pygame.draw.rect(self.image, (220, 220, 220), (8, 8, w - 16, 10), border_radius=3)
        self.rect = self.image.get_rect(center=(x, y))
        self.speed = speed

    def update(self):
        self.rect.y += self.speed
        if self.rect.top > HEIGHT + 40:
            self.kill()

class Player(Car):
    def __init__(self):
        super().__init__(lane_x(LANE_COUNT // 2), HEIGHT - 100, PLAYER_WIDTH, PLAYER_HEIGHT, PLAYER_COLOR)
        self.vx = 0
        self.vy = 0

    def handle_input(self, keys):
        self.vx = 0
        self.vy = 0
        if keys[pygame.K_LEFT]:
            self.vx = -PLAYER_SPEED
        if keys[pygame.K_RIGHT]:
            self.vx = PLAYER_SPEED
        if keys[pygame.K_UP]:
            self.vy = -PLAYER_SPEED
        if keys[pygame.K_DOWN]:
            self.vy = PLAYER_SPEED

    def update(self):
        self.rect.x += self.vx
        self.rect.y += self.vy
        # keep within road bounds
        left_bound = ROAD_MARGIN + 10
        right_bound = WIDTH - ROAD_MARGIN - 10 - self.rect.width
        self.rect.x = clamp(self.rect.x, left_bound, right_bound)
        self.rect.y = clamp(self.rect.y, 120, HEIGHT - 20)

# --- Game state ---
player = Player()
all_sprites = pygame.sprite.Group(player)
enemies = pygame.sprite.Group()

score = 0
running = True
game_over = False
paused = False

SPAWN_EVENT = pygame.USEREVENT + 1
pygame.time.set_timer(SPAWN_EVENT, SPAWN_INTERVAL_MS)

road_scroll = 0
lane_line_gap = 40

def spawn_enemy():
    lane = random.randrange(LANE_COUNT)
    x = lane_x(lane)
    y = -ENEMY_HEIGHT
    speed = random.randint(ENEMY_MIN_SPEED, ENEMY_MAX_SPEED)
    enemy = Car(x, y, ENEMY_WIDTH, ENEMY_HEIGHT, ENEMY_COLOR, speed)
    # slight random horizontal offset within lane
    enemy.rect.centerx += random.randint(-LANE_WIDTH // 4, LANE_WIDTH // 4)
    enemies.add(enemy)
    all_sprites.add(enemy)

def reset():
    global player, all_sprites, enemies, score, game_over, paused, road_scroll
    all_sprites.empty()
    enemies.empty()
    player = Player()
    all_sprites.add(player)
    score = 0
    game_over = False
    paused = False
    road_scroll = 0

def draw_road(surface):
    # road background
    pygame.draw.rect(surface, ROAD, (ROAD_MARGIN, 0, WIDTH - 2 * ROAD_MARGIN, HEIGHT))
    # lane dividers (scrolling dashed lines)
    global road_scroll
    road_scroll = (road_scroll + 4) % lane_line_gap
    for i in range(LANE_COUNT - 1):
        x = ROAD_MARGIN + (i + 1) * LANE_WIDTH
        for y in range(-lane_line_gap, HEIGHT, lane_line_gap * 2):
            pygame.draw.line(surface, LINE, (x, y + road_scroll), (x, y + road_scroll + lane_line_gap), 3)
    # roadside edges
    pygame.draw.rect(surface, LINE, (ROAD_MARGIN - 6, 0, 4, HEIGHT))
    pygame.draw.rect(surface, LINE, (WIDTH - ROAD_MARGIN + 2, 0, 4, HEIGHT))

def draw_hud(surface):
    score_surf = font.render(f"Score: {score}", True, TEXT)
    surface.blit(score_surf, (16, 16))
    if paused:
        p = big_font.render("PAUSED", True, (255, 220, 120))
        surface.blit(p, (WIDTH // 2 - p.get_width() // 2, 40))

def draw_game_over(surface):
    title = big_font.render("Game Over", True, (255, 120, 120))
    hint = font.render("Press R to restart or Q to quit", True, TEXT)
    best = font.render(f"Final score: {score}", True, TEXT)
    surface.blit(title, (WIDTH // 2 - title.get_width() // 2, HEIGHT // 2 - 60))
    surface.blit(best, (WIDTH // 2 - best.get_width() // 2, HEIGHT // 2))
    surface.blit(hint, (WIDTH // 2 - hint.get_width() // 2, HEIGHT // 2 + 40))

# --- Main loop ---
while running:
    dt = clock.tick(FPS) / 1000.0

    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False
        elif event.type == SPAWN_EVENT and not (paused or game_over):
            spawn_enemy()
        elif event.type == pygame.KEYDOWN:
            if event.key == pygame.K_p and not game_over:
                paused = not paused
            if game_over:
                if event.key == pygame.K_r:
                    reset()
                elif event.key == pygame.K_q:
                    running = False

    keys = pygame.key.get_pressed()
    if not (paused or game_over):
        player.handle_input(keys)
        all_sprites.update()
        # collision
        if pygame.sprite.spritecollideany(player, enemies):
            game_over = True
        # score increases with time and speed of enemies
        score += 1

    # --- Render ---
    screen.fill(BG)
    draw_road(screen)
    all_sprites.draw(screen)
    draw_hud(screen)
    if game_over:
        draw_game_over(screen)
    pygame.display.flip()

pygame.quit()
sys.exit()
