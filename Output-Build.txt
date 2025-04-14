#include <iostream>
#include <vector>
#include <string>
#include <conio.h>
#include <chrono>
#include <thread>
#include <algorithm>
#include <cstdlib>
#include <ctime>

using namespace std;
using namespace std::chrono;

const int screenWidth = 40;
const int screenHeight = 15;
const int worldWidth = 100;

enum GameState { TITLE, PLAYING, PAUSED, VICTORY };
GameState gameState = TITLE;

struct Bullet { int x, y; bool fromPlayer; };
struct Item { string name; int x, y; };

struct Player {
    int x = 2, y = screenHeight - 2, vy = 0, health = 3, maxHealth = 3;
    bool onGround = true, canDash = true;
    int dashDistance = 5, shootCooldown = 1000;
    time_point<steady_clock> lastDash = steady_clock::now();
    time_point<steady_clock> lastShot = steady_clock::now();
    vector<string> inventory;
};

struct Enemy { int x, y, hp = 2, vy = 0; bool onGround = true; };

struct Boss {
    int x = worldWidth - 10, y = screenHeight - 2, hp = 10;
    bool alive = true;
};

vector<string> world(screenHeight, string(worldWidth, ' '));
vector<Bullet> bullets;
vector<Enemy> enemies;
vector<Item> items;
Player player;
Boss boss;
int worldOffset = 0;

void showTitleScreen() {
    system("cls");
    cout << "=====================================\n";
    cout << "         WELCOME TO ASCII QUEST      \n";
    cout << "=====================================\n";
    cout << "\n   Controls:\n";
    cout << "   WASD - Move/Jump\n";
    cout << "   X - Shoot | C - Dash\n";
    cout << "   P - Pause | Q - Quit\n";
    cout << "\n[Press ENTER to Start]\n";
    while (_getch() != '\r');
    gameState = PLAYING;
}

void showPauseMenu() {
    system("cls");
    cout << "=== PAUSED ===\n";
    cout << "Press 'R' to Resume or 'Q' to Quit\n";
    while (true) {
        if (_kbhit()) {
            char key = _getch();
            if (key == 'r') {
                gameState = PLAYING;
                break;
            }
            if (key == 'q') exit(0);
        }
    }
}

void showVictoryScreen() {
    system("cls");
    cout << "\n============================\n";
    cout << "     CONGRATULATIONS!      \n";
    cout << "     YOU DEFEATED BOSS     \n";
    cout << "============================\n";
    cout << "[Press Q to Quit]\n";
    while (_getch() != 'q');
    exit(0);
}

void draw() {
    vector<string> screen(screenHeight, string(screenWidth, ' '));

    for (int y = 0; y < screenHeight; y++) {
        for (int x = 0; x < screenWidth; x++) {
            int worldX = x + worldOffset;
            if (worldX >= 0 && worldX < worldWidth)
                screen[y][x] = world[y][worldX];
        }
    }

    for (auto& b : bullets) {
        int sx = b.x - worldOffset;
        if (b.y >= 0 && b.y < screenHeight && sx >= 0 && sx < screenWidth)
            screen[b.y][sx] = b.fromPlayer ? '-' : '*';
    }

    for (auto& e : enemies) {
        int sx = e.x - worldOffset;
        if (e.y >= 0 && e.y < screenHeight && sx >= 0 && sx < screenWidth)
            screen[e.y][sx] = 'E';
    }

    if (boss.alive) {
        int sx = boss.x - worldOffset;
        if (boss.y >= 0 && boss.y < screenHeight && sx >= 0 && sx < screenWidth)
            screen[boss.y][sx] = 'B';
    }

    for (auto& i : items) {
        int sx = i.x - worldOffset;
        if (i.y >= 0 && i.y < screenHeight && sx >= 0 && sx < screenWidth)
            screen[i.y][sx] = '$';
    }

    int px = player.x - worldOffset;
    if (player.y >= 0 && player.y < screenHeight && px >= 0 && px < screenWidth)
        screen[player.y][px] = '@';

    system("cls");
    for (auto& row : screen) cout << row << '\n';

    cout << "HP: " << player.health << "/" << player.maxHealth << " | Inventory: ";
    for (auto& item : player.inventory) cout << "[" << item << "] ";

    if (boss.alive) {
        cout << " | Boss HP: [";
        for (int i = 0; i < boss.hp; i++) cout << "#";
        for (int i = boss.hp; i < 10; i++) cout << " ";
        cout << "] " << boss.hp << "/10";
    }

    cout << "\nUse WASD, X=Shoot, C=Dash, P=Pause, Q=Quit\n";
}

void updateWorld() {
    for (int y = 0; y < screenHeight; y++)
        for (int x = 0; x < worldWidth; x++)
            world[y][x] = ' ';

    for (int x = 0; x < worldWidth; x++)
        world[screenHeight - 1][x] = '=';

    for (int x = 10; x < 20; x++) world[10][x] = '=';
    for (int x = 25; x < 30; x++) world[8][x] = '=';
    for (int x = 35; x < 40; x++) world[6][x] = '=';
    for (int x = 50; x < 60; x++) world[9][x] = '=';
    for (int x = 65; x < 70; x++) world[7][x] = '=';
    for (int x = 75; x < 85; x++) world[5][x] = '=';
    for (int x = 90; x < 95; x++) world[10][x] = '=';
}

bool isGround(int x, int y) {
    return x >= 0 && x < worldWidth && y + 1 < screenHeight && world[y + 1][x] == '=';
}

void updatePlayer() {
    if (!player.onGround) {
        player.vy++;
        player.y += player.vy;
        if (player.y >= screenHeight - 2 || isGround(player.x, player.y)) {
            player.y = screenHeight - 2;
            player.vy = 0;
            player.onGround = true;
        }
    }

    if (player.x - worldOffset > screenWidth / 2 && worldOffset < worldWidth - screenWidth)
        worldOffset++;

    if (_kbhit()) {
        char key = _getch();
        if (key == 'a' && player.x > 0) player.x--;
        if (key == 'd' && player.x < worldWidth - 1) player.x++;
        if (key == 'w' && player.onGround) {
            player.vy = -3;
            player.onGround = false;
        }
        if (key == 'x') {
            auto now = steady_clock::now();
            if (duration_cast<milliseconds>(now - player.lastShot).count() > player.shootCooldown) {
                bullets.push_back({ player.x + 1, player.y, true });
                player.lastShot = now;
            }
        }
        if (key == 'c' && player.canDash) {
            player.x = min(player.x + player.dashDistance, worldWidth - 1);
            player.canDash = false;
            player.lastDash = steady_clock::now();
        }
        if (key == 'p') gameState = PAUSED;
        if (key == 'q') exit(0);
    }

    if (!player.canDash &&
        duration_cast<milliseconds>(steady_clock::now() - player.lastDash).count() > 2000)
        player.canDash = true;
}

void updateBullets() {
    for (auto& b : bullets) b.x += b.fromPlayer ? 1 : -1;

    for (auto& b : bullets) {
        if (b.fromPlayer && boss.alive && b.x == boss.x && b.y == boss.y) {
            boss.hp--;
            b.x = -1000;
            if (boss.hp <= 0) {
                boss.alive = false;
                gameState = VICTORY;
            }
        }
    }

    bullets.erase(remove_if(bullets.begin(), bullets.end(),
        [](Bullet& b) { return b.x < 0 || b.x >= worldWidth; }), bullets.end());
}

void updateEnemies() {
    for (auto& e : enemies) {
        if (e.x > player.x) e.x--;
        else if (e.x < player.x) e.x++;

        if (!isGround(e.x, e.y)) {
            if (e.onGround) { e.vy = -2; e.onGround = false; }
        }

        if (!e.onGround) {
            e.vy++;
            e.y += e.vy;
            if (e.y >= screenHeight - 2 || isGround(e.x, e.y)) {
                e.y = screenHeight - 2;
                e.vy = 0; e.onGround = true;
            }
        }
    }
}

int main() {
    srand(time(0));
    updateWorld();
    enemies.push_back({ 20, screenHeight - 2 });
    enemies.push_back({ 28, 9 });
    enemies.push_back({ 36, 5 });
    enemies.push_back({ 52, 8 });
    enemies.push_back({ 67, 6 });
    enemies.push_back({ 76, 4 });

    while (true) {
        switch (gameState) {
        case TITLE: showTitleScreen(); break;
        case PLAYING:
            draw();
            updatePlayer();
            updateBullets();
            updateEnemies();
            this_thread::sleep_for(milliseconds(100));
            break;
        case PAUSED: showPauseMenu(); break;
        case VICTORY: showVictoryScreen(); break;
        }
    }

    return 0;
}
﻿
