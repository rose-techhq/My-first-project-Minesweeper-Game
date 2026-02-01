#include <iostream>
#include <vector>
#include <ctime>
#include <cstdlib>
#include <limits>
#include <string>
#include <cmath>
#include <chrono>
#include <fstream>

// ANSI Escape Codes for console coloring
#define RESET   "\033[0m"
#define PINK    "\033[35m"
#define GREEN   "\033[32m"
#define RED     "\033[31m"
#define CYAN    "\033[36m"
#define YELLOW  "\033[33m"

// Structure to represent a single tile on the board
struct Cell {
    bool isMine = false;        // True if the cell contains a mine
    bool isRevealed = false;    // True if the player has opened the cell
    bool isFlagged = false;     // True if the player marked it with a flag
    int adjacentMines = 0;      // Number of mines in the surrounding cells
};

class Minesweeper {
private:
    int size;                   // Grid dimensions (size x size)
    int totalMines;             // Total number of mines hidden in the grid
    bool hitMine = false;       // Flag to track if the player triggered a mine
    std::vector<std::vector<Cell>> board; // 2D vector representing the game board
    std::chrono::steady_clock::time_point startTime; // Tracks start time for the timer
    bool timerStarted = false;  // Flag to ensure the timer starts on the first move

    // Checks if a given coordinate (row, col) is within the board boundaries
    bool isValid(int r, int c) { return (r >= 0 && r < size && c >= 0 && c < size); }

    // Scans the board and calculates the 'adjacentMines' count for every non-mine cell
    void calculateNumbers() {
        for (int r = 0; r < size; r++) {
            for (int c = 0; c < size; c++) {
                board[r][c].adjacentMines = 0;
                if (board[r][c].isMine) continue;
                // Check all 8 neighbors
                for (int i = -1; i <= 1; i++) {
                    for (int j = -1; j <= 1; j++) {
                        if (isValid(r + i, c + j) && board[r + i][c + j].isMine) {
                            board[r][c].adjacentMines++;
                        }
                    }
                }
            }
        }
    }

public:
    // Constructor: Initializes board, plants mines (15% of grid), and shows briefing
    Minesweeper(int s) : size(s) {
        totalMines = static_cast<int>(std::floor((s * s) * 0.15));
        if (totalMines < 1) totalMines = 1;
        board.resize(size, std::vector<Cell>(size));

        // Randomly plant mines across the board
        int planted = 0;
        while (planted < totalMines) {
            int r = rand() % size;
            int c = rand() % size;
            if (!board[r][c].isMine) {
                board[r][c].isMine = true;
                planted++;
            }
        }
        calculateNumbers();

        // Print initial Mission Briefing to the user
        std::cout << PINK << "\n[ MISSION BRIEFING ]" << RESET << "\n";
        std::cout << YELLOW << "📊 Grid Size: " << size << "x" << size << "\n";
        std::cout << RED << "💣 Total Mines: " << totalMines << RESET << "\n";
        std::cout << CYAN << "--------------------------" << RESET << "\n";
    }

    // Records the exact moment the first action is taken
    void startTimer() {
        if (!timerStarted) {
            startTime = std::chrono::steady_clock::now();
            timerStarted = true;
        }
    }

    // Calculates and returns time spent in seconds since start
    int getElapsedSeconds() {
        if (!timerStarted) return 0;
        auto now = std::chrono::steady_clock::now();
        return std::chrono::duration_cast<std::chrono::seconds>(now - startTime).count();
    }

    // Main logic for revealing a cell; includes recursion
    void reveal(int r, int c, bool isSilent = false) {
        if (!isValid(r, c) || board[r][c].isRevealed || board[r][c].isFlagged) return;

        startTimer();
        board[r][c].isRevealed = true;

        if (board[r][c].isMine) { hitMine = true; return; }

        // If cell has no neighboring mines, automatically reveal neighbors (Flood Fill)
        if (board[r][c].adjacentMines == 0) {
            for (int i = -1; i <= 1; i++) {
                for (int j = -1; j <= 1; j++) {
                    if (i != 0 || j != 0) reveal(r + i, c + j, true);
                }
            }
        }
    }

    // Allows the user to place or remove a flag on a hidden cell
    void toggleFlag(int r, int c) {
        if (!isValid(r, c) || board[r][c].isRevealed) return;
        startTimer();
        board[r][c].isFlagged = !board[r][c].isFlagged;
        std::cout << YELLOW << (board[r][c].isFlagged ? "🚩 Flagged " : "🔓 Unflagged ")
                  << "(" << r << "," << c << ")" << RESET << "\n";
    }

    // Writes the current board state to a text file for later recovery
    void saveGame() {
        std::ofstream outFile("minesweeper_save.txt");
        outFile << size << " " << totalMines << "\n";
        for (int r = 0; r < size; r++) {
            for (int c = 0; c < size; c++) {
                outFile << board[r][c].isMine << " " << board[r][c].isRevealed << " " << board[r][c].isFlagged << " ";
            }
            outFile << "\n";
        }
        outFile.close();
        std::cout << GREEN << "💾 Game saved successfully!" << RESET << "\n";
    }

    // Reads from the save file and reconstructs the game state
    bool loadGame() {
        std::ifstream inFile("minesweeper_save.txt");
        if (!inFile) return false;
        inFile >> size >> totalMines;
        board.assign(size, std::vector<Cell>(size));
        for (int r = 0; r < size; r++) {
            for (int c = 0; c < size; c++) {
                inFile >> board[r][c].isMine >> board[r][c].isRevealed >> board[r][c].isFlagged;
            }
        }
        inFile.close();
        calculateNumbers();
        timerStarted = false; // Reset timer for loaded game
        return true;
    }

    // Prints the ASCII representation of the board to the terminal
    void display(bool revealAll = false) {
        std::cout << "\n    ";
        for (int i = 0; i < size; i++) std::cout << PINK << "  " << i << (i < 10 ? " " : "") << RESET;
        std::cout << "\n";
        for (int r = 0; r < size; r++) {
            std::cout << "    +";
            for (int i = 0; i < size; i++) std::cout << "---+";
            std::cout << "\n";
            std::cout << PINK << (r < 10 ? " " : "") << r << "  " << RESET << "|";
            for (int c = 0; c < size; c++) {
                if (board[r][c].isFlagged && !revealAll) std::cout << " 🚩|";
                else if (!board[r][c].isRevealed && !revealAll) std::cout << GREEN << " ■ " << RESET << "|";
                else if (board[r][c].isMine) std::cout << " 💣|";
                else if (board[r][c].adjacentMines == 0) std::cout << "   |";
                else {
                    int n = board[r][c].adjacentMines;
                    std::cout << " ";
                    if (n == 1) std::cout << CYAN << n << RESET << " |";
                    else if (n == 2) std::cout << GREEN << n << RESET << " |";
                    else std::cout << RED << n << RESET << " |";
                }
            }
            std::cout << "\n";
        }
        std::cout << "    +";
        for (int i = 0; i < size; i++) std::cout << "---+";
        std::cout << "\n";
    }

    // Checks if all non-mine cells have been revealed
    bool checkWin() {
        for (int r = 0; r < size; r++) {
            for (int c = 0; c < size; c++) {
                if (!board[r][c].isMine && !board[r][c].isRevealed) return false;
            }
        }
        return true;
    }
    bool hasLost() { return hitMine; }
};

// Helper function to handle non-integer input gracefully
int getNumberInput(const std::string& prompt) {
    int value;
    while (true) {
        std::cout << YELLOW << prompt << RESET;
        if (std::cin >> value) return value;
        std::cout << RED << "❌ Error: Numbers only!" << RESET << "\n";
        std::cin.clear();
        std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
    }
}

// Program execution entry point
int main() {
    srand(static_cast<unsigned int>(time(0))); // Seed for random number generation
    bool globalRunning = true; // Controls the "Play Again" loop

    while (globalRunning) {
        std::cout << YELLOW << "=======================================\n";
        std::cout << "    🎮 WELCOME TO MINESWEEPER 💣🚩    \n";
        std::cout << "=======================================\n" << RESET;

        // Ask user to start a new game or resume from a save
        char startChoice;
        while (true) {
            std::cout << YELLOW << "[n] 🆕 New Game   [l] 📂 Load Last Save: " << RESET;
            std::cin >> startChoice;
            if (startChoice == 'n' || startChoice == 'N' || startChoice == 'l' || startChoice == 'L') break;
            std::cout << RED << "❌ Error: Use 'n' or 'l' only." << RESET << "\n";
            std::cin.clear(); std::cin.ignore(1000, '\n');
        }

        Minesweeper* game;
        if (startChoice == 'l' || startChoice == 'L') {
            game = new Minesweeper(5); // Temp instance to call load
            if (!game->loadGame()) {
                std::cout << RED << "❌ Error: No save file found! Starting new.\n" << RESET;
                delete game;
                game = new Minesweeper(getNumberInput("Enter grid size: "));
            } else {
                std::cout << GREEN << "📂 Game Loaded successfully!" << RESET << "\n";
            }
        } else {
            game = new Minesweeper(getNumberInput("Enter grid size: "));
        }

        bool gameInProgress = true;
        while (gameInProgress) {
            game->display();
            std::cout << YELLOW << "Actions: [r] 🔍 Reveal [f] 🚩 Flag/Unflag [s] 💾 Save [q] 🚪 Quit\nSelection: " << RESET;
            char action; std::cin >> action;

            if (action == 's') { game->saveGame(); continue; }
            if (action == 'q') { gameInProgress = false; break; }

            if (action == 'r' || action == 'f') {
                int r = getNumberInput("Row: ");
                int c = getNumberInput("Col: ");
                if (action == 'r') game->reveal(r, c);
                else if (action == 'f') game->toggleFlag(r, c);
            } else {
                std::cout << RED << "❌ Error: Invalid action!" << RESET << "\n";
            }

            // End-game condition checks
            if (game->hasLost()) {
                game->display(true);
                std::cout << RED << "\n💥 BOOM! 💥\n😭 Game Over! Better luck next time!\n⏱ Time Taken: " << game->getElapsedSeconds() << "s\n" << RESET;
                gameInProgress = false;
            } else if (game->checkWin()) {
                game->display(true);
                std::cout << GREEN << "\n🏆 WINNER! 🏆\n⏱ Time Taken: " << game->getElapsedSeconds() << "s\n" << RESET;
                gameInProgress = false;
            }
        }

        delete game; // Prevent memory leaks

        // Ask for a restart
        char playAgain;
        while (true) {
            std::cout << YELLOW << "\n🔄 Play again? [y] Yes  [n] No: " << RESET;
            std::cin >> playAgain;
            if (playAgain == 'y' || playAgain == 'Y') break;
            else if (playAgain == 'n' || playAgain == 'N') {
                globalRunning = false;
                std::cout << YELLOW << "\n👋 Thanks for playing! See you next time.\n" << RESET;
                break;
            } else {
                std::cout << RED << "❌ Error: Use 'y' or 'n' only." << RESET << "\n";
                std::cin.clear(); std::cin.ignore(1000, '\n'); // dicard bad characters in the buffer
            }
        }
    }
    return 0;
}
