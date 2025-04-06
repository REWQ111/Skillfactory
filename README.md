import random

class Cell:  
    def __init__(self, row, col):
        self.row = row
        self.col = col

    def __eq__(self, other):
        return self.row == other.row and self.col == other.col

    def __hash__(self):
        return hash((self.row, self.col))

    def __repr__(self):
        return f"({self.row}, {self.col})"


class SeaError(Exception):
    pass


class SeaBoardOut(SeaError):
    def __str__(self):
        return "Попытка выстрела за пределы поля!"


class SeaBoardUsed(SeaError):
    def __str__(self):
        return "Сюда уже стреляли!"


class SeaShipError(Exception):
    pass


class NavalVessel:
    def __init__(self, start_coord, length, direction):
        self.start_coord = start_coord
        self.length = length
        self.direction = direction
        self.health = length
    @property
    def coords(self):
        ship_cells = []
        for i in range(self.length):
            x_coord = self.start_coord.row
            y_coord = self.start_coord.col

            if self.direction == 0:
                x_coord += i
            elif self.direction == 1:
                y_coord += i

            ship_cells.append(Cell(x_coord, y_coord))

        return ship_cells

    def hit(self, shot):
        return shot in self.coords


class Ocean:
    def __init__(self, hidden=False, size=6):
        self.size = size
        self.hidden = hidden

        self.destroyed = 0

        self.grid = [["O"] * size for _ in range(size)]
        self.occupied = set()
        self.fleet = []

    def add_ship(self, ship):
        for cell in ship.coords:
            if self.out_of_bounds(cell) or cell in self.occupied:
                raise SeaShipError()

        for cell in ship.coords:
            self.grid[cell.row][cell.col] = "■"
            self.occupied.add(cell)

        self.fleet.append(ship)
        self.mark_area(ship)

    def mark_area(self, ship):
        area = [
            (-1, -1), (-1, 0), (-1, 1),
            (0, -1), (0, 0), (0, 1),
            (1, -1), (1, 0), (1, 1)
        ]
        for cell in ship.coords:
            for x_offset, y_offset in area:
                neighbor = Cell(cell.row + x_offset, y_offset + cell.col)
                if not (self.out_of_bounds(neighbor)) and neighbor not in self.occupied:
                    self.occupied.add(neighbor)

    def __str__(self):
        display = ""
        display += "  | 1 | 2 | 3 | 4 | 5 | 6 |"
        for i, row in enumerate(self.grid):
            display += f"\n{i + 1} | " + " | ".join(row) + " |"

        if self.hidden:
            display = display.replace("■", "O")  # Скрываем корабли на поле компьютера

        return display

    def out_of_bounds(self, cell):
        return not ((0 <= cell.row < self.size) and (0 <= cell.col < self.size))

    def attempt_shot(self, shot):
        if self.out_of_bounds(shot):
            raise SeaBoardOut()

        if shot in self.occupied and (self.grid[shot.row][shot.col] == "T" or self.grid[shot.row][shot.col] == "X"):  # Сюда нельзя, если это T или X
            raise SeaBoardUsed()

        for ship in self.fleet:
            if shot in ship.coords:
                ship.health -= 1
                self.grid[shot.row][shot.col] = "X"
                if ship.health == 0:  #
                    self.destroyed += 1
                    self.mark_area(ship)
                    print("Корабль уничтожен!")
                    return False
                else:
                    print("Корабль поврежден!")
                    return True

        self.grid[shot.row][shot.col] = "T"
        self.occupied.add(shot)
        print("Мимо!")
        return False

class Controller:
    def __init__(self, sea, opponent):
        self.sea = sea
        self.opponent = opponent

    def ask_shot(self):
        raise NotImplementedError()

    def make_move(self):
        while True:
            try:
                target = self.ask_shot()
                repeat = self.opponent.attempt_shot(target)
                return repeat
            except SeaError as e:
                print(e)

class Brain(Controller):
    def ask_shot(self):
        while True:
            target = Cell(random.randint(0, 5), random.randint(0, 5))
            try:
                self.sea.attempt_shot(target) # Пробуем выстрелить, чтобы проверить занятость
                print(f"Ход компьютера: {target.row + 1} {target.col + 1}")
                return target
            except SeaBoardUsed:
                continue

class Human(Controller):
    def ask_shot(self):
        while True:
            coords = input("Ваш выстрел (x y): ").split()

            if len(coords) != 2:
                print("Введите 2 координаты!")
                continue

            try:
                x, y = map(int, coords)
                return Cell(x - 1, y - 1)
            except ValueError:
                print("Введите числа!")

class Flow:  # Игра
    def __init__(self, size=6):
        self.size = size
        self.ship_lengths = [3, 2, 2, 1, 1, 1]
        self.player_sea = self.create_random_fleet()
        self.comp_sea = self.create_random_fleet()
        self.comp_sea.hidden = True

        self.computer = Brain(self.comp_sea, self.player_sea)
        self.human = Human(self.player_sea, self.comp_sea)
        self.total_ships = len(self.ship_lengths)

    def create_random_fleet(self):
        ocean = Ocean(size=self.size)
        attempts = 0
        for length in self.ship_lengths:
            while True:
                attempts += 1
                if attempts > 2000:
                    return None
                ship = NavalVessel(Cell(random.randint(0, self.size - 1), random.randint(0, self.size - 1)), length, random.randint(0, 1))
                try:
                    ocean.add_ship(ship)
                    break
                except SeaShipError:
                    pass
        return ocean

    def show_info(self):
        print("Приветствуем в игре")
        print("Морской бой!")
        print("Формат ввода: x y")
        print("x - номер строки")
        print("y - номер столбца")

    def game_process(self):
        turn_count = 0
        while True:
            print("-" * 20)
            print("Ваша доска:")
            print(self.human.sea)
            print("-" * 20)
            print("Доска противника:")
            print(self.computer.sea)

            if turn_count % 2 == 0:
                print("-" * 20)
                print("Ваш ход!")
                repeat = self.human.make_move()
            else:
                print("-" * 20)
                print("Ходит компьютер!")
                repeat = self.computer.make_move()

            if repeat:
                turn_count -= 1
            if self.comp_sea.destroyed == self.total_ships:
                print("-" * 20)
                print("Вы победили!")
                break

            if self.player_sea.destroyed == self.total_ships:
                print("-" * 20)
                print("Компьютер выиграл!")
                break

            turn_count += 1

    def begin(self):
        self.show_info()
        self.game_process()


game = Flow()
game.begin()
