#include <SFML/Graphics.hpp>
#include <vector>
#include <ctime>
#include <cstdlib>
#include <iostream>
#include <sstream>
#include <random>

const int GRID_SIZE = 4;
const int CARD_WIDTH = 80;
const int CARD_HEIGHT = 80;
int WINDOW_WIDTH = GRID_SIZE * CARD_WIDTH + 200;
int WINDOW_HEIGHT = GRID_SIZE * CARD_HEIGHT;

class Card {
public:
    sf::RectangleShape shape, border;
    sf::Color originalColor;
    bool flipped;

    Card(int x, int y, sf::Color color) {
        shape.setSize(sf::Vector2f(CARD_WIDTH, CARD_HEIGHT));
        shape.setPosition(x, y);
        shape.setFillColor(sf::Color(169, 169, 169));

        border.setSize(sf::Vector2f(CARD_WIDTH, CARD_HEIGHT));
        border.setPosition(x, y);
        border.setFillColor(sf::Color::Transparent);
        border.setOutlineThickness(4);
        border.setOutlineColor(sf::Color::Black);

        flipped = false;
        originalColor = color;
    }

    void flip() {
        flipped = !flipped;
        shape.setFillColor(flipped ? originalColor : sf::Color(169, 169, 169));
    }

    void draw(sf::RenderWindow &window) {
        window.draw(shape);
        window.draw(border);
    }
};

void initializeCards(std::vector<Card> &cards, std::vector<sf::Color> &colors) {
    cards.clear();
    int numPairs = (GRID_SIZE * GRID_SIZE) / 2;
    colors.clear();

    for (int i = 0; i < numPairs; ++i) {
        colors.push_back(sf::Color(rand() % 256, rand() % 256, rand() % 256));
    }
    colors.insert(colors.end(), colors.begin(), colors.end());
    std::random_shuffle(colors.begin(), colors.end());

    int index = 0;
    for (int i = 0; i < GRID_SIZE; ++i) {
        for (int j = 0; j < GRID_SIZE; ++j) {
            cards.emplace_back(j * CARD_WIDTH, i * CARD_HEIGHT, colors[index]);
            index++;
        }
    }
}

void displayMessage(sf::RenderWindow &window, const std::string &message, sf::Font &font) {
    sf::Text text(message, font, 30);
    text.setFillColor(sf::Color::White);
    text.setPosition(WINDOW_WIDTH / 2 - text.getLocalBounds().width / 2, WINDOW_HEIGHT / 2 - 30);
    window.clear();
    window.draw(text);
    window.display();
}

void displayCardSequence(sf::RenderWindow &window, std::vector<Card> &cards, int viewTime) {
    for (auto &card : cards) {
        card.flip();
    }
    window.clear(sf::Color::Black);
    for (auto &card : cards) {
        card.draw(window);
    }
    window.display();
    sf::sleep(sf::seconds(viewTime));
    for (auto &card : cards) {
        card.flip();
    }
}

void displayLevelComplete(sf::RenderWindow &window, sf::Font &font) {
    displayMessage(window, "Well done!", font);
    sf::sleep(sf::seconds(2));
    displayMessage(window, "Prepare for next level!", font);
    sf::sleep(sf::seconds(2));
}

void displayGameOver(sf::RenderWindow &window, sf::Font &font) {
    displayMessage(window, "Game Over! Try again!", font);
    sf::sleep(sf::seconds(2));
}

std::string getRandomComment(int score, int mistakes) {
    std::vector<std::string> positive = {"You're on fire!", "Amazing memory!", "Keep it up!"};
    std::vector<std::string> negative = {"Oops, focus harder!", "Don't give up!", "You can do this!"};
    return mistakes > score / 10 ? negative[rand() % negative.size()] : positive[rand() % positive.size()];
}

int main() {
    int level = 1, score = 0, mistakes = 0, hardshipLevel = 1;
    int cardShowTime = 5;

    sf::RenderWindow window(sf::VideoMode(WINDOW_WIDTH, WINDOW_HEIGHT), "Memory Matching Game");
    sf::Font font;
    if (!font.loadFromFile("C:\\Windows\\Fonts\\arial.ttf")) {
        std::cerr << "Failed to load font!\n";
        return -1;
    }

    std::vector<Card> cards;
    std::vector<sf::Color> colors;

    initializeCards(cards, colors);
    bool firstCardFlipped = false, secondCardFlipped = false;
    int firstCardIndex = -1, secondCardIndex = -1;
    bool gameStarted = false;

    displayMessage(window, "Click to Start!", font);
    while (window.isOpen() && !gameStarted) {
        sf::Event event;
        while (window.pollEvent(event)) {
            if (event.type == sf::Event::Closed) window.close();
            if (event.type == sf::Event::MouseButtonPressed && event.mouseButton.button == sf::Mouse::Left) {
                gameStarted = true;
            }
        }
    }

    std::string currentComment = "Get ready!";  // Initial comment
    while (window.isOpen()) {
        int viewTime = std::max(3, 6 - hardshipLevel);

        displayCardSequence(window, cards, viewTime);

        sf::Clock levelClock;

        while (window.isOpen()) {
            sf::Event event;
            while (window.pollEvent(event)) {
                if (event.type == sf::Event::Closed) window.close();

                if (event.type == sf::Event::MouseButtonPressed && event.mouseButton.button == sf::Mouse::Left) {
                    sf::Vector2i mousePos = sf::Mouse::getPosition(window);
                    int x = mousePos.x / CARD_WIDTH;
                    int y = mousePos.y / CARD_HEIGHT;

                    if (x < GRID_SIZE && y < GRID_SIZE) {
                        int cardIndex = y * GRID_SIZE + x;

                        if (!cards[cardIndex].flipped && gameStarted) {
                            cards[cardIndex].flip();
                            if (!firstCardFlipped) {
                                firstCardFlipped = true;
                                firstCardIndex = cardIndex;
                            } else if (!secondCardFlipped) {
                                secondCardFlipped = true;
                                secondCardIndex = cardIndex;
                            }
                        }
                    }
                }
            }

            if (firstCardFlipped && secondCardFlipped) {
                if (cards[firstCardIndex].originalColor == cards[secondCardIndex].originalColor) {
                    score += 10 * level;
                    currentComment = getRandomComment(score, mistakes);  // Update comment after a correct match
                    firstCardFlipped = secondCardFlipped = false;

                    bool allFlipped = true;
                    for (auto &card : cards) {
                        if (!card.flipped) {
                            allFlipped = false;
                            break;
                        }
                    }

                    if (allFlipped) {
                        displayLevelComplete(window, font);
                        level++;
                        if (level % 3 == 0) hardshipLevel++;

                        initializeCards(cards, colors);
                        levelClock.restart();
                        break;
                    }
                } else {
                    sf::sleep(sf::seconds(0.5));
                    cards[firstCardIndex].flip();
                    cards[secondCardIndex].flip();
                    score -= 2 * level;
                    mistakes++;
                    currentComment = getRandomComment(score, mistakes);  // Update comment after a mistake
                    firstCardFlipped = secondCardFlipped = false;

                    if (score <= -20 || mistakes >= 10) {
                        displayGameOver(window, font);
                        return 0;
                    }
                }
            }

            window.clear(sf::Color::Black);
            for (auto &card : cards) card.draw(window);

            sf::Text scoreText("Score: " + std::to_string(score), font, 20);
            scoreText.setPosition(WINDOW_WIDTH - 180, 20);
            window.draw(scoreText);

            sf::Text levelText("Level: " + std::to_string(level), font, 20);
            levelText.setPosition(WINDOW_WIDTH - 180, 50);
            window.draw(levelText);

            sf::Text hardshipText("Hardship Level: " + std::to_string(hardshipLevel), font, 20);
            hardshipText.setPosition(WINDOW_WIDTH - 180, 80);
            window.draw(hardshipText);

            sf::Text mistakeText("Mistakes: " + std::to_string(mistakes), font, 20);
            mistakeText.setPosition(WINDOW_WIDTH - 180, 110);
            window.draw(mistakeText);

            int remainingTime = 105 - levelClock.getElapsedTime().asSeconds();
            std::stringstream timeStream;
            timeStream << "Time: " << remainingTime << "s";
            sf::Text timerText(timeStream.str(), font, 20);
            timerText.setPosition(WINDOW_WIDTH - 180, 140);
            window.draw(timerText);

            // Add a two-line gap before the "Cheers:" heading
            sf::Text cheersHeading("Cheers:", font, 20);
            cheersHeading.setPosition(WINDOW_WIDTH - 180, 170 + 40); // Adjusted gap (two-line gap)
            cheersHeading.setFillColor(sf::Color::White); // White for heading
            window.draw(cheersHeading);

            // Display the comment (below "Cheers:" heading)
            sf::Text funnyComment(currentComment, font, 18);
            funnyComment.setFillColor(sf::Color::Cyan);
            funnyComment.setPosition(WINDOW_WIDTH - 180, 200 + 40); // Adjusted positioning
            window.draw(funnyComment);

            window.display();

            if (remainingTime <= 0) {
                displayMessage(window, "Time's Up! Game Over!", font);
                sf::sleep(sf::seconds(2));
                return 0;
            }

            if (level > 10) {
                displayMessage(window, "YOUR MEMORY IS A MIRACLE", font);
                sf::sleep(sf::seconds(2));
                return 0;
            }
        }
    }

    return 0;
}
