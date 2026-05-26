/*******************************************************************************************
*
*   raylib - classic game: tetris (C++ Version)
*
********************************************************************************************/
#include "raylib.h"
#include <cstdio>
#include <cstdlib>
#include <ctime>
#include <cmath>

#if defined(PLATFORM_WEB)
    #include <emscripten/emscripten.h>
#endif

//----------------------------------------------------------------------------------
// 常量定义
//----------------------------------------------------------------------------------
const int SQUARE_SIZE = 20;

const int GRID_HORIZONTAL_SIZE = 12;
const int GRID_VERTICAL_SIZE = 20;

const int LATERAL_SPEED = 10;
const int TURNING_SPEED = 12;
const int FAST_FALL_AWAIT_COUNTER = 30;

const int FADING_TIME = 33;

//----------------------------------------------------------------------------------
// 枚举定义
//----------------------------------------------------------------------------------
enum class GridSquare { EMPTY, MOVING, FULL, BLOCK, FADING };

//------------------------------------------------------------------------------------
// 全局变量
//------------------------------------------------------------------------------------
static const int screenWidth = 800;
static const int screenHeight = 450;

static bool gameOver = false;
static bool pause = false;

// 矩阵
static GridSquare grid[GRID_HORIZONTAL_SIZE][GRID_VERTICAL_SIZE];
static GridSquare piece[4][4];
static GridSquare incomingPiece[4][4];

// 当前方块位置
static int piecePositionX = 0;
static int piecePositionY = 0;

static Color fadingColor;
static bool beginPlay = true;
static bool pieceActive = false;
static bool detection = false;
static bool lineToDelete = false;

// 游戏数据
static int level = 1;
static int lines = 0;
static int score = 0;

// 计数器
static int gravityMovementCounter = 0;
static int lateralMovementCounter = 0;
static int turnMovementCounter = 0;
static int fastFallMovementCounter = 0;

static int fadeLineCounter = 0;

static int gravitySpeed = 30;

//------------------------------------------------------------------------------------
// 函数声明
//------------------------------------------------------------------------------------
static void InitGame(void);
static void UpdateGame(void);
static void DrawGame(void);
static void UnloadGame(void);
static void UpdateDrawFrame(void);

static bool CreatePiece();
static void GetRandomPiece();
static void ResolveFallingMovement(bool* detection, bool* pieceActive);
static bool ResolveLateralMovement();
static bool ResolveTurnMovement();
static void CheckDetection(bool* detection);
static void CheckCompletion(bool* lineToDelete);
static void DeleteCompleteLines();

//------------------------------------------------------------------------------------
// 主函数
//------------------------------------------------------------------------------------
int main(void)
{
    InitWindow(screenWidth, screenHeight, "classic game: tetris [C++]");

    InitGame();

#if defined(PLATFORM_WEB)
    emscripten_set_main_loop(UpdateDrawFrame, 60, 1);
#else
    SetTargetFPS(60);
    while (!WindowShouldClose())
    {
        UpdateDrawFrame();
    }
#endif

    UnloadGame();
    CloseWindow();

    return 0;
}

//--------------------------------------------------------------------------------------
// 游戏初始化
//--------------------------------------------------------------------------------------
void InitGame(void)
{
    level = 1;
    lines = 0;
    score = 0;

    fadingColor = GRAY;

    piecePositionX = 0;
    piecePositionY = 0;

    pause = false;
    beginPlay = true;
    pieceActive = false;
    detection = false;
    lineToDelete = false;

    gravityMovementCounter = 0;
    lateralMovementCounter = 0;
    turnMovementCounter = 0;
    fastFallMovementCounter = 0;

    fadeLineCounter = 0;
    gravitySpeed = 30;

    // 初始化网格
    for (int i = 0; i < GRID_HORIZONTAL_SIZE; i++)
    {
        for (int j = 0; j < GRID_VERTICAL_SIZE; j++)
        {
            if ((j == GRID_VERTICAL_SIZE - 1) || (i == 0) || (i == GRID_HORIZONTAL_SIZE - 1))
                grid[i][j] = GridSquare::BLOCK;
            else
                grid[i][j] = GridSquare::EMPTY;
        }
    }

    // 初始化下一个方块
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            incomingPiece[i][j] = GridSquare::EMPTY;
        }
    }
}

//--------------------------------------------------------------------------------------
// 游戏逻辑更新
//--------------------------------------------------------------------------------------
void UpdateGame(void)
{
    if (!gameOver)
    {
        if (IsKeyPressed(KEY_P)) pause = !pause;

        if (!pause)
        {
            if (!lineToDelete)
            {
                if (!pieceActive)
                {
                    pieceActive = CreatePiece();
                    fastFallMovementCounter = 0;
                }
                else
                {
                    fastFallMovementCounter++;
                    gravityMovementCounter++;
                    lateralMovementCounter++;
                    turnMovementCounter++;

                    if (IsKeyPressed(KEY_LEFT) || IsKeyPressed(KEY_RIGHT))
                        lateralMovementCounter = LATERAL_SPEED;
                    if (IsKeyPressed(KEY_UP))
                        turnMovementCounter = TURNING_SPEED;

                    if (IsKeyDown(KEY_DOWN) && (fastFallMovementCounter >= FAST_FALL_AWAIT_COUNTER))
                    {
                        gravityMovementCounter += gravitySpeed;
                    }

                    if (gravityMovementCounter >= gravitySpeed)
                    {
                        CheckDetection(&detection);
                        ResolveFallingMovement(&detection, &pieceActive);
                        CheckCompletion(&lineToDelete);
                        gravityMovementCounter = 0;
                    }

                    if (lateralMovementCounter >= LATERAL_SPEED)
                    {
                        if (!ResolveLateralMovement())
                            lateralMovementCounter = 0;
                    }

                    if (turnMovementCounter >= TURNING_SPEED)
                    {
                        if (ResolveTurnMovement())
                            turnMovementCounter = 0;
                    }
                }

                // 游戏结束判定
                for (int j = 0; j < 2; j++)
                {
                    for (int i = 1; i < GRID_HORIZONTAL_SIZE - 1; i++)
                    {
                        if (grid[i][j] == GridSquare::FULL)
                        {
                            gameOver = true;
                        }
                    }
                }
            }
            else
            {
                fadeLineCounter++;
                if (fadeLineCounter % 8 < 4) fadingColor = MAROON;
                else fadingColor = GRAY;

                if (fadeLineCounter >= FADING_TIME)
                {
                    DeleteCompleteLines();
                    fadeLineCounter = 0;
                    lineToDelete = false;
                    lines++;
                    score += 100;

                    // 等级提升
                    if (lines % 5 == 0 && gravitySpeed > 10)
                    {
                        level++;
                        gravitySpeed -= 5;
                    }
                }
            }
        }
    }
    else
    {
        if (IsKeyPressed(KEY_ENTER))
        {
            InitGame();
            gameOver = false;
        }
    }
}

//--------------------------------------------------------------------------------------
// 游戏绘制
//--------------------------------------------------------------------------------------
void DrawGame(void)
{
    BeginDrawing();
        ClearBackground(RAYWHITE);

        if (!gameOver)
        {
            Vector2 offset;
            offset.x = screenWidth/2 - (GRID_HORIZONTAL_SIZE*SQUARE_SIZE/2) - 50;
            offset.y = screenHeight/2 - ((GRID_VERTICAL_SIZE - 1)*SQUARE_SIZE/2) + SQUARE_SIZE*2;
            offset.y -= 50;

            int controller = offset.x;

            // 绘制主游戏区
            for (int j = 0; j < GRID_VERTICAL_SIZE; j++)
            {
                for (int i = 0; i < GRID_HORIZONTAL_SIZE; i++)
                {
                    if (grid[i][j] == GridSquare::EMPTY)
                    {
                        DrawLine(offset.x, offset.y, offset.x + SQUARE_SIZE, offset.y, LIGHTGRAY);
                        DrawLine(offset.x, offset.y, offset.x, offset.y + SQUARE_SIZE, LIGHTGRAY);
                        DrawLine(offset.x + SQUARE_SIZE, offset.y, offset.x + SQUARE_SIZE, offset.y + SQUARE_SIZE, LIGHTGRAY);
                        DrawLine(offset.x, offset.y + SQUARE_SIZE, offset.x + SQUARE_SIZE, offset.y + SQUARE_SIZE, LIGHTGRAY);
                    }
                    else if (grid[i][j] == GridSquare::FULL)
                    {
                        DrawRectangle(offset.x, offset.y, SQUARE_SIZE, SQUARE_SIZE, GRAY);
                    }
                    else if (grid[i][j] == GridSquare::MOVING)
                    {
                        DrawRectangle(offset.x, offset.y, SQUARE_SIZE, SQUARE_SIZE, DARKGRAY);
                    }
                    else if (grid[i][j] == GridSquare::BLOCK)
                    {
                        DrawRectangle(offset.x, offset.y, SQUARE_SIZE, SQUARE_SIZE, LIGHTGRAY);
                    }
                    else if (grid[i][j] == GridSquare::FADING)
                    {
                        DrawRectangle(offset.x, offset.y, SQUARE_SIZE, SQUARE_SIZE, fadingColor);
                    }
                    offset.x += SQUARE_SIZE;
                }
                offset.x = controller;
                offset.y += SQUARE_SIZE;
            }

            // 绘制下一个方块预览
            offset.x = 500;
            offset.y = 45;
            int controler = offset.x;

            for (int j = 0; j < 4; j++)
            {
                for (int i = 0; i < 4; i++)
                {
                    if (incomingPiece[i][j] == GridSquare::EMPTY)
                    {
                        DrawLine(offset.x, offset.y, offset.x + SQUARE_SIZE, offset.y, LIGHTGRAY);
                        DrawLine(offset.x, offset.y, offset.x, offset.y + SQUARE_SIZE, LIGHTGRAY);
                        DrawLine(offset.x + SQUARE_SIZE, offset.y, offset.x + SQUARE_SIZE, offset.y + SQUARE_SIZE, LIGHTGRAY);
                        DrawLine(offset.x, offset.y + SQUARE_SIZE, offset.x + SQUARE_SIZE, offset.y + SQUARE_SIZE, LIGHTGRAY);
                    }
                    else if (incomingPiece[i][j] == GridSquare::MOVING)
                    {
                        DrawRectangle(offset.x, offset.y, SQUARE_SIZE, SQUARE_SIZE, GRAY);
                    }
                    offset.x += SQUARE_SIZE;
                }
                offset.x = controler;
                offset.y += SQUARE_SIZE;
            }

            DrawText("INCOMING:", offset.x, offset.y - 100, 10, GRAY);
            DrawText(TextFormat("LINES:  %04i", lines), offset.x, offset.y + 20, 10, GRAY);
            DrawText(TextFormat("LEVEL:  %02i", level), offset.x, offset.y + 40, 10, GRAY);
            DrawText(TextFormat("SCORE:  %06i", score), offset.x, offset.y + 60, 10, GRAY);

            if (pause)
                DrawText("GAME PAUSED", screenWidth/2 - MeasureText("GAME PAUSED", 40)/2, screenHeight/2 - 40, 40, GRAY);
        }
        else
        {
            DrawText("GAME OVER", screenWidth/2 - MeasureText("GAME OVER", 60)/2, screenHeight/2 - 80, 60, RED);
            DrawText("PRESS [ENTER] TO PLAY AGAIN", screenWidth/2 - MeasureText("PRESS [ENTER] TO PLAY AGAIN", 20)/2, screenHeight/2, 20, GRAY);
        }

    EndDrawing();
}

void UnloadGame(void) { }

void UpdateDrawFrame(void)
{
    UpdateGame();
    DrawGame();
}

//--------------------------------------------------------------------------------------
// 方块生成
//--------------------------------------------------------------------------------------
bool CreatePiece()
{
    piecePositionX = (GRID_HORIZONTAL_SIZE - 4)/2;
    piecePositionY = 0;

    if (beginPlay)
    {
        GetRandomPiece();
        beginPlay = false;
    }

    for (int i = 0; i < 4; i++)
        for (int j = 0; j < 4; j++)
            piece[i][j] = incomingPiece[i][j];

    GetRandomPiece();

    for (int i = piecePositionX; i < piecePositionX + 4; i++)
        for (int j = 0; j < 4; j++)
            if (piece[i - piecePositionX][j] == GridSquare::MOVING)
                grid[i][j] = GridSquare::MOVING;

    return true;
}

void GetRandomPiece()
{
    int random = GetRandomValue(0, 6);

    for (int i = 0; i < 4; i++)
        for (int j = 0; j < 4; j++)
            incomingPiece[i][j] = GridSquare::EMPTY;

    switch (random)
    {
        case 0: // 正方形
            incomingPiece[1][1] = GridSquare::MOVING;
            incomingPiece[2][1] = GridSquare::MOVING;
            incomingPiece[1][2] = GridSquare::MOVING;
            incomingPiece[2][2] = GridSquare::MOVING;
            break;
        case 1: // L型
            incomingPiece[1][0] = GridSquare::MOVING;
            incomingPiece[1][1] = GridSquare::MOVING;
            incomingPiece[1][2] = GridSquare::MOVING;
            incomingPiece[2][2] = GridSquare::MOVING;
            break;
        case 2: // 反L型
            incomingPiece[1][2] = GridSquare::MOVING;
            incomingPiece[2][0] = GridSquare::MOVING;
            incomingPiece[2][1] = GridSquare::MOVING;
            incomingPiece[2][2] = GridSquare::MOVING;
            break;
        case 3: // 直线
            incomingPiece[0][1] = GridSquare::MOVING;
            incomingPiece[1][1] = GridSquare::MOVING;
            incomingPiece[2][1] = GridSquare::MOVING;
            incomingPiece[3][1] = GridSquare::MOVING;
            break;
        case 4: // T型
            incomingPiece[1][0] = GridSquare::MOVING;
            incomingPiece[1][1] = GridSquare::MOVING;
            incomingPiece[1][2] = GridSquare::MOVING;
            incomingPiece[2][1] = GridSquare::MOVING;
            break;
        case 5: // S型
            incomingPiece[1][1] = GridSquare::MOVING;
            incomingPiece[2][1] = GridSquare::MOVING;
            incomingPiece[2][2] = GridSquare::MOVING;
            incomingPiece[3][2] = GridSquare::MOVING;
            break;
        case 6: // 反S型
            incomingPiece[1][2] = GridSquare::MOVING;
            incomingPiece[2][2] = GridSquare::MOVING;
            incomingPiece[2][1] = GridSquare::MOVING;
            incomingPiece[3][1] = GridSquare::MOVING;
            break;
    }
}

//--------------------------------------------------------------------------------------
// 移动、旋转、碰撞、消行逻辑
//--------------------------------------------------------------------------------------
void ResolveFallingMovement(bool* detection, bool* pieceActive)
{
    if (*detection)
    {
        for (int j = GRID_VERTICAL_SIZE - 2; j >= 0; j--)
        {
            for (int i = 1; i < GRID_HORIZONTAL_SIZE - 1; i++)
            {
                if (grid[i][j] == GridSquare::MOVING)
                {
                    grid[i][j] = GridSquare::FULL;
                    *detection = false;
                    *pieceActive = false;
                }
            }
        }
    }
    else
    {
        for (int j = GRID_VERTICAL_SIZE - 2; j >= 0; j--)
        {
            for (int i = 1; i < GRID_HORIZONTAL_SIZE - 1; i++)
            {
                if (grid[i][j] == GridSquare::MOVING)
                {
                    grid[i][j+1] = GridSquare::MOVING;
                    grid[i][j] = GridSquare::EMPTY;
                }
            }
        }
        piecePositionY++;
    }
}

bool ResolveLateralMovement()
{
    bool collision = false;

    if (IsKeyDown(KEY_LEFT))
    {
        for (int j = GRID_VERTICAL_SIZE - 2; j >= 0; j--)
            for (int i = 1; i < GRID_HORIZONTAL_SIZE - 1; i++)
                if (grid[i][j] == GridSquare::MOVING)
                    if ((i-1 == 0) || (grid[i-1][j] == GridSquare::FULL))
                        collision = true;

        if (!collision)
        {
            for (int j = GRID_VERTICAL_SIZE - 2; j >= 0; j--)
                for (int i = 1; i < GRID_HORIZONTAL_SIZE - 1; i++)
                    if (grid[i][j] == GridSquare::MOVING)
                    {
                        grid[i-1][j] = GridSquare::MOVING;
                        grid[i][j] = GridSquare::EMPTY;
                    }
            piecePositionX--;
        }
    }
    else if (IsKeyDown(KEY_RIGHT))
    {
        for (int j = GRID_VERTICAL_SIZE - 2; j >= 0; j--)
            for (int i = 1; i < GRID_HORIZONTAL_SIZE - 1; i++)
                if (grid[i][j] == GridSquare::MOVING)
                    if ((i+1 == GRID_HORIZONTAL_SIZE - 1) || (grid[i+1][j] == GridSquare::FULL))
                        collision = true;

        if (!collision)
        {
            for (int j = GRID_VERTICAL_SIZE - 2; j >= 0; j--)
                for (int i = GRID_HORIZONTAL_SIZE - 1; i >= 1; i--)
                    if (grid[i][j] == GridSquare::MOVING)
                    {
                        grid[i+1][j] = GridSquare::MOVING;
                        grid[i][j] = GridSquare::EMPTY;
                    }
            piecePositionX++;
        }
    }
    return collision;
}

bool ResolveTurnMovement()
{
    if (IsKeyDown(KEY_UP))
    {
        GridSquare tempPiece[4][4];
        for (int i = 0; i < 4; i++)
            for (int j = 0; j < 4; j++)
                tempPiece[i][j] = piece[i][j];

        // 旋转矩阵
        for (int i = 0; i < 4; i++)
            for (int j = 0; j < 4; j++)
                piece[i][j] = tempPiece[3 - j][i];

        // 碰撞检测
        bool canRotate = true;
        for (int i = 0; i < 4; i++)
        {
            for (int j = 0; j < 4; j++)
            {
                if (piece[i][j] == GridSquare::MOVING)
                {
                    int x = piecePositionX + i;
                    int y = piecePositionY + j;
                    if (x < 1 || x >= GRID_HORIZONTAL_SIZE - 1 || y >= GRID_VERTICAL_SIZE - 1 || grid[x][y] == GridSquare::FULL)
                    {
                        canRotate = false;
                        break;
                    }
                }
            }
            if (!canRotate) break;
        }

        if (!canRotate)
        {
            for (int i = 0; i < 4; i++)
                for (int j = 0; j < 4; j++)
                    piece[i][j] = tempPiece[i][j];
            return false;
        }

        // 清除旧位置
        for (int j = 0; j < GRID_VERTICAL_SIZE - 1; j++)
            for (int i = 1; i < GRID_HORIZONTAL_SIZE - 1; i++)
                if (grid[i][j] == GridSquare::MOVING)
                    grid[i][j] = GridSquare::EMPTY;

        // 绘制新位置
        for (int i = 0; i < 4; i++)
            for (int j = 0; j < 4; j++)
                if (piece[i][j] == GridSquare::MOVING)
                    grid[piecePositionX + i][piecePositionY + j] = GridSquare::MOVING;

        return true;
    }
    return false;
}

void CheckDetection(bool* detection)
{
    *detection = false;
    for (int j = GRID_VERTICAL_SIZE - 2; j >= 0; j--)
    {
        for (int i = 1; i < GRID_HORIZONTAL_SIZE - 1; i++)
        {
            if (grid[i][j] == GridSquare::MOVING &&
               (grid[i][j+1] == GridSquare::FULL || grid[i][j+1] == GridSquare::BLOCK))
            {
                *detection = true;
                return;
            }
        }
    }
}

void CheckCompletion(bool* lineToDelete)
{
    *lineToDelete = false;
    for (int j = GRID_VERTICAL_SIZE - 2; j >= 0; j--)
    {
        int count = 0;
        for (int i = 1; i < GRID_HORIZONTAL_SIZE - 1; i++)
        {
            if (grid[i][j] == GridSquare::FULL) count++;
        }

        if (count == GRID_HORIZONTAL_SIZE - 2)
        {
            *lineToDelete = true;
            for (int z = 1; z < GRID_HORIZONTAL_SIZE - 1; z++)
                grid[z][j] = GridSquare::FADING;
        }
    }
}

void DeleteCompleteLines()
{
    for (int j = GRID_VERTICAL_SIZE - 2; j >= 0; j--)
    {
        if (grid[1][j] == GridSquare::FADING)
        {
            for (int i = 1; i < GRID_HORIZONTAL_SIZE - 1; i++)
                grid[i][j] = GridSquare::EMPTY;

            for (int j2 = j - 1; j2 >= 0; j2--)
            {
                for (int i2 = 1; i2 < GRID_HORIZONTAL_SIZE - 1; i2++)
                {
                    if (grid[i2][j2] == GridSquare::FULL)
                    {
                        grid[i2][j2 + 1] = GridSquare::FULL;
                        grid[i2][j2] = GridSquare::EMPTY;
                    }
                }
            }
        }
    }
}
