# projekt3.1

/*
* Ein verbessertes Labyrinth-Spiel
 * Autor: Fritz Bökler (fboekler@uos.de)
 * Datum: 02.12.2024
 * MIT Lizenz
 *
 * In diesem Spiel versucht eine SpielerIn (S) das Ziel (Z) zu erreichen.
 * Das Labyrinth wird ueber die Konsole (cout) ausgegeben, die Eingabe erfolgt ebenfalls
 * zeilengepuffert ueber die Konsole (cin).
 *
 * Das Labyrinth enthält die folgenden Zeichen:
 * . Leeres Feld
 * # Wand (nicht begehbar)
 * Z Ziel
 * S SpielerIn (wird nicht im Labyrint selbst gespeichert)
 * K Schluessel
 * T Tür
 * A Geist
 *
 * Eine SpielerIn hat eine Anzahl an Schlüsseln. Diese Anzahl wird beim Erreichen eines
 * K-Feldes erhöht und beim Erreichen eines T-Feldes reduziert. Eine Tür kann nur durchschritten
 * werden, wenn die SpielerIn mindestens einen Schluessel besitzt. Ein aufgenommener Schluessel
 * verschwindet (wird zu .), ebenso wie eine durchschrittene Tuer.
 *
 * Die folgenden Eingaben sind gültig:
 * w - nach oben bewegen
 * a - nach links bewegen
 * s - nach unten bewegen
 * d - nach rechts bewegen
 * q - Spiel beenden
 *
 * Das Labyrnith wird zu Beginn eingegeben.
 * Syntax: <Zeilen> <Spalte> <Labyrinth-Zeichen> <Spieler Zeile> <Spieler Spalte>
 *   <Zeilen>: 1 bis 20
 *   <Spalten>: 1 bis 20
 *   <Labyrint-Zeichen>: Eine Folge von <Zeilen> * <Spalten> vielen Zeichen aus {., #, Z, K, T, A}
 *   <Spieler Zeile>: 0 bis <Zeilen> - 1
 *   <Spieler Spalte>: 0 bis <Spalten> - 1
 *
 * Ein Beispiellabyrinth: 7 7 ...#....#...#T..####Z#....##K###.#......A#.###### 0 4
*/

#include "std_lib_inc.h"

// Exception fuer nicht erlaubte Bewegungen
class BadMovement {};

// Exception fuer unbekannte Eingaben
class UnknownInput {};

// Exception fuer eine falsche Labyrinth-Eingabe
class BadMaze {};

// Klasse, die eine SpielerIn kapselt
class Player
{
public:
    int no_keys; // Anzahl der Schlüssel der SpielerIn
    vector<int> position; // Aktuelle Position der SpielerIn im Labyrinth
};

// Klasse, die das Labyrinth kapselt
class Maze
{
public:
    int rows; // Anzahl der Zeilen des Labyrinths
    int cols; // Anzahl der Spalten des Labyrinths
    vector<vector<char>> data; // Labyrinth-Daten (erst Zeilen dann Spalten)
    vector<int> player_start_position; // Startposition der SpielerIn, so wie es in der Übergabe steht
};

// Fasst Labyrinth und Spieler zu einem Spiel-Status zusammen
class GameState
{
public:
    Maze maze; // Das Labyrinth
    Player player; // Die SpielerIn
    bool exit; // Wurde 'q' gerdückt?
    bool hit_ghost; // Wurde ein Geist getroffen?
};

// Funktion zur Anzeige des Spielfeldes
void display_maze(GameState game_state)
{
    const int player_row = game_state.player.position[0];
    const int player_col = game_state.player.position[1];

    //cout << "\033[H\033[J"; // ANSI Escape Code zum Loeschen des Bildschirms
    for(int i = 0; i < game_state.maze.rows; i++)
    {
        for(int j = 0; j < game_state.maze.cols; j++)
        {
            if(i == player_row && j == player_col)
            {
                cout << 'S';
            }
            else
            {
                cout << game_state.maze.data[i][j];
            }
            cout << " ";
        }
        cout << '\n';
    }
}

// Funktion zur Umrechnung eines Kommandos zu einer neuen Position
// Vorbedingung: direction muss aus {w, s, a, d} kommen.
vector<int> new_position_by_direction(vector<int> player_position, char direction)
{
    const int row = player_position[0];
    const int col = player_position[1];

    switch(direction)
    {
        case 'w':
            return {row - 1, col};
        case 's':
            return {row + 1, col};
        case 'a':
            return {row, col - 1};
        case 'd':
            return {row, col + 1};
        default:
            assert(false, "new_position_by_direction: invalid direction, assumes direction is one of {w, s, a, d}.");
            return {};
    }
}

// Fuehrt Aktionen des Spieler-Feldes aus
// Vorbedingung: Wenn das Feld eine Tuer ist, muss mindestens ein Schluessel zur Verfuegung stehen
GameState process_tile_action(GameState game_state)
{
    const int row = game_state.player.position[0];
    const int col = game_state.player.position[1];

    assert(game_state.maze.data[row][col] != 'T' || game_state.player.no_keys > 0,
        "process_tile_action(...) assumes enough keys are there when approaching a door.");

    if(game_state.maze.data[row][col] == 'K')
    {
        ++game_state.player.no_keys;
        game_state.maze.data[row][col] = '.';
    }
    else if(game_state.maze.data[row][col] == 'T')
    {
        --game_state.player.no_keys;
        game_state.maze.data[row][col] = '.';
    }
    else if(game_state.maze.data[row][col] == 'A')
    {
        game_state.hit_ghost = true;
    }
    return game_state;
}

// Gibt true zurueck gdw. die Position begehbar ist
bool position_is_walkable(vector<int> position, GameState game_state)
{
    const int row = position[0];
    const int col = position[1];

    if(row < 0 || col < 0)
    {
        return false;
    }
    if(row >= game_state.maze.rows || col >= game_state.maze.cols)
    {
        return false;
    }
    if(game_state.maze.data[row][col] == '#')
    {
        return false;
    }
    if(game_state.maze.data[row][col] == 'T' && game_state.player.no_keys == 0)
    {
        return false;
    }
    return true;
}

// Funktion zur Bewegung der SpielerIn
GameState move_player(GameState game_state, char direction)
{
    vector<int> potential_new_position = new_position_by_direction(game_state.player.position, direction);

    if(!position_is_walkable(potential_new_position, game_state))
    {
        throw BadMovement {};
    }

    game_state.player.position = potential_new_position;
    return process_tile_action(game_state);
}

// Gibt eine kurze Hilfe aus
void display_help()
{
    cout << "Willkommen zum Labyrinth-Spiel!\n";
    cout << "Ziel des Spiels: Finden Sie den Weg vom Startpunkt (S) zum Ziel (Z).\n";
    cout << "Spielfeld-Erklaerung:\n";
    cout << "S - Startpunkt: Hier beginnt die SpielerIn.\n";
    cout << "Z - Ziel: Erreichen Sie diesen Punkt, um das Spiel zu gewinnen.\n";
    cout << "# - Wand: Diese Felder sind nicht begehbar.\n";
    cout << "K - Schluessel: Hier können Sie einen Schluessel aufsammeln, um eine Tuer zu oeffnen.\n";
    cout << "T - Tuer: Unbegehbares Feld, ausser, Sie haben einen Schluessel. Beim Durchschreiten wird ein Schluessel verbraucht.\n";
    cout << "A - Geist: Ein Geist im Labyrinth. Stehen die SpielerIn auf dem selben Feld, verlieren Sie das Spiel!\n";
    cout << ". - Leeres Feld: Diese Felder koennen betreten werden.\n";
    cout << "\nSteuerung:\n";
    cout << "w - Nach oben bewegen\n";
    cout << "a - Nach links bewegen\n";
    cout << "s - Nach unten bewegen\n";
    cout << "d - Nach rechts bewegen\n";
    cout << "q - Spiel beenden\n";
    cout << "Nach jeder Befehlseingabe muss die Eingabetaste (Enter) gedrueckt werden, um sich zu bewegen.\n";
    cout << "\nViel Erfolg im Labyrinth!\n";
}

// Reagiert auf das eingegebene Kommando und gibt an die jeweilige Funktion
// ab, die sich um genau dieses Kommando kuemmert.
GameState process_input(GameState game_state, char input)
{
    switch(input)
    {
        case 'w':
        case 's':
        case 'a':
        case 'd':
            return move_player(game_state, input);
        case 'h':
        case 'H':
            display_help();
            break;
        case 'q':
            game_state.exit = true;
            return game_state;
        default:
            throw UnknownInput{};
    }
    return game_state;
}

// Gibt true zurueck, wenn das Ziel erreicht wurde
bool reached_goal(GameState game_state)
{
    return game_state.maze.data[game_state.player.position[0]][game_state.player.position[1]] == 'Z';
}

// Gibt true zurueck gdw der Geist getroffen wurde
bool hit_ghost(GameState game_state)
{
    return game_state.hit_ghost;
}

// Gibt true zurueck gdw. das Spiel zuende ist
bool is_end_condition(GameState game_state)
{
    return reached_goal(game_state) || hit_ghost(game_state) || game_state.exit;
}

// Die Hauptschleife des Spiels
GameState game_loop(GameState game_state)
{
    char input;
    while(cin && !is_end_condition(game_state))
    {
        assert(game_state.player.no_keys >= 0,
            "Player has a negative number of keys.");

        display_maze(game_state);

        cin >> input;
        if(cin)
        {
            try
            {
                game_state = process_input(game_state, input);
            }
            catch(BadMovement&)
            {
                cout << "Bewegung nicht moeglich!\n";
            }
            catch(UnknownInput&)
            {
                cout << "Diese Eingabe kenne ich nicht. Gib 'h' ein, um eine Hilfe zu erhalten.\n";
            }
        }
    }

    return game_state;
}

// Liest ein integer von der Eingabe.
// Vorbedingung: cin ist in Ordnung
int read_int()
{
    int integer;
    cin >> integer;
    if(!cin) {throw BadMaze{};}
    return integer;
}

// Liest die Labyrinth-Daten ein.
// Vorbedingung: cin ist ok.
vector<vector<char>> read_maze_data(int rows, int cols)
{
    vector<vector<char>> maze_data(rows);
    char ch;
    for(int i = 0; i < rows * cols; ++i)
    {
        cin >> ch;
        if(!cin) {throw BadMaze {};}
        if(!(ch == '#' || ch == 'T' || ch == 'A' || ch == '.' || ch == 'K' || ch == 'Z'))
        {
            throw BadMaze {};
        }

        maze_data[i / cols].push_back(ch);
    }
    return maze_data;
}

// Liest das Labyrinth von der Konsole nach der Formatdefinition aus der Aufgabe
Maze read_maze()
{
    int rows = read_int();
    int cols = read_int();

    if(rows < 1 || cols < 1 || rows > 20 || cols > 20)
    {
        throw BadMaze {};
    }

    vector<vector<char>> labyrinth_data = read_maze_data(rows, cols);

    int player_row = read_int();
    int player_col = read_int();

    if(player_row < 0 || player_row >= rows || player_col < 0 || player_col >= cols)
    {
        throw BadMaze {};
    }

    if(labyrinth_data[player_row][player_col] != '.')
    {
        throw BadMaze {};
    }

    return {rows, cols, labyrinth_data, {player_row, player_col}};
}

// Initialisiert das Labyrinth-Objekt
GameState initialize()
{
    Maze maze = read_maze();
    Player player{0, maze.player_start_position};

    return GameState {maze, player, false, false};
}

int main()
{
    activateAssertions();
    try
    {
        GameState game_state = initialize();

        game_state = game_loop(game_state);

        if(reached_goal(game_state))
        {
            display_maze(game_state);
            cout << "Ziel erreicht! Herzlichen Glueckwunsch!\n";
        }
        else if(hit_ghost(game_state))
        {
            cout << "Sie haben einen Geist getroffen! Game Over!\n";
        }
        else
        {
            cout << "Schoenen Tag noch!\n";
        }
        return 0;
    }
    catch(BadMaze&)
    {
        cout << "Fehler beim Einlesen des Labyrinths.\n";
        return 1;
    }
    catch(...)
    {
        cout << "Unbekannter Fehler!\n";
        return 1;
    }
}
