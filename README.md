#include <iostream>
#include <vector>
#include <string>
#include <memory>
#include <exception>
#include <iomanip>
#include <cstdlib>
using namespace std;

class AppException : public exception {
protected:
    string msg;
public:
    explicit AppException(const string& m) : msg(m) {}
    const char* what() const noexcept override { return msg.c_str(); }
};
class InvalidInputException : public AppException {
public: explicit InvalidInputException(const string& m) : AppException("[Invalid input] " + m) {}
};
class NotFoundException : public AppException {
public: explicit NotFoundException(const string& m) : AppException("[Not found] " + m) {}
};
class OrderStateException : public AppException {
public: explicit OrderStateException(const string& m) : AppException("[Order state] " + m) {}
};
class PaymentException : public AppException {
public: explicit PaymentException(const string& m) : AppException("[Payment] " + m) {}
};

string money(double v) {
    long long x = (long long)(v + 0.5);
    string s = to_string(x), r;
    int c = 0;
    for (int i = (int)s.size() - 1; i >= 0; --i) {
        r = s[i] + r;
        if (++c % 3 == 0 && i > 0) r = "," + r;
    }
    return r + " VND";
}

class Person {
protected:
    int id;
    string name, phone;
    static int totalPersons;
public:
    Person(int id = 0, const string& name = "", const string& phone = "") : id(id) {
        setName(name);
        setPhone(phone);
        ++totalPersons;
    }
    virtual ~Person() { --totalPersons; }

    int getId() const { return id; }
    const string& getName() const { return name; }
    const string& getPhone() const { return phone; }
    void setName(const string& n) {
        if (n.empty()) throw InvalidInputException("Name must not be empty");
        name = n;
    }
    void setPhone(const string& p) {
        if (p.size() < 9 || p.size() > 11) throw InvalidInputException("Phone must have 9-11 digits");
        for (char c : p) if (c < '0' || c > '9') throw InvalidInputException("Phone must contain digits only");
        phone = p;
    }
    static int getTotalPersons() { return totalPersons; }

    virtual string role() const = 0;
    virtual void display() const = 0;
};
int Person::totalPersons = 0;

class Customer : virtual public Person {
protected:
    string address;
    int points;
public:
    Customer(int id, const string& name, const string& phone, const string& addr)
        : Person(id, name, phone), address(addr), points(0) {}
    const string& getAddress() const { return address; }
    int getPoints() const { return points; }
    void addPoints(double total) { points += (int)(total / 10000); }
    virtual double discountRate() const { return 0.0; }
    string role() const override { return "Customer"; }
    void display() const override {
        cout << "  [" << role() << "] #" << id << " " << name << " | " << phone
             << " | " << address << " | points: " << points
             << " | discount: " << discountRate() * 100 << "%\n";
    }
};

class VIPCustomer : public Customer {
public:
    VIPCustomer(int id, const string& name, const string& phone, const string& addr)
        : Person(id, name, phone), Customer(id, name, phone, addr) {}
    double discountRate() const override { return 0.10; }
    string role() const override { return "VIP Customer"; }
};

class Driver : virtual public Person {
protected:
    string plate;
    bool available;
public:
    Driver(int id, const string& name, const string& phone, const string& plate)
        : Person(id, name, phone), plate(plate), available(true) {}
    bool isAvailable() const { return available; }
    void setAvailable(bool a) { available = a; }
    string role() const override { return "Driver"; }
    void display() const override {
        cout << "  [" << role() << "] #" << id << " " << name << " | " << phone
             << " | plate: " << plate << " | " << (available ? "available" : "busy") << "\n";
    }
};

class Admin : public Person {
public:
    Admin(int id, const string& name, const string& phone) : Person(id, name, phone) {}
    string role() const override { return "Admin"; }
    void display() const override {
        cout << "  [" << role() << "] #" << id << " " << name << " | " << phone << "\n";
    }
};

class DriverCustomer : public Customer, public Driver {
public:
    DriverCustomer(int id, const string& name, const string& phone, const string& addr, const string& plate)
        : Person(id, name, phone), Customer(id, name, phone, addr), Driver(id, name, phone, plate) {}
    string role() const override { return "Driver & Customer"; }
    void display() const override {
        cout << "  [" << role() << "] #" << id << " " << name << " | " << phone
             << " | " << address << " | plate: " << plate << "\n";
    }
};

class MenuItem {
protected:
    string name;
    double basePrice;
public:
    MenuItem(const string& n, double p) : name(n), basePrice(p) {
        if (p <= 0) throw InvalidInputException("Price must be positive");
    }
    virtual ~MenuItem() {}
    const string& getName() const { return name; }
    virtual double getPrice() const = 0;
    virtual string describe() const = 0;
    friend ostream& operator<<(ostream& os, const MenuItem& m) {
        return os << m.describe() << " - " << money(m.getPrice());
    }
};
class Food : public MenuItem {
    bool spicy;
public:
    Food(const string& n, double p, bool spicy = false) : MenuItem(n, p), spicy(spicy) {}
    double getPrice() const override { return basePrice; }
    string describe() const override { return "Food: " + name + (spicy ? " (spicy)" : ""); }
};
class Drink : public MenuItem {
    char size;
public:
    Drink(const string& n, double p, char size = 'M') : MenuItem(n, p), size(size) {
        if (size != 'S' && size != 'M' && size != 'L') throw InvalidInputException("Drink size must be S/M/L");
    }
    double getPrice() const override { return basePrice * (size == 'S' ? 1.0 : size == 'M' ? 1.2 : 1.5); }
    string describe() const override { return string("Drink: ") + name + " (" + size + ")"; }
};
class Combo : public MenuItem {
    vector<shared_ptr<MenuItem>> items;
public:
    Combo(const string& n, const vector<shared_ptr<MenuItem>>& its) : MenuItem(n, 1), items(its) {
        if (its.empty()) throw InvalidInputException("Combo must contain at least one item");
    }
    double getPrice() const override {
        double s = 0;
        for (auto& i : items) s += i->getPrice();
        return s * 0.85;
    }
    string describe() const override { return "Combo: " + name + " (" + to_string(items.size()) + " items, -15%)"; }
};

class Payment {
public:
    virtual ~Payment() {}
    virtual double fee(double amount) const = 0;
    virtual void pay(double amount) = 0;
    virtual string method() const = 0;
};
class CashPayment : public Payment {
public:
    double fee(double) const override { return 0; }
    void pay(double) override {}
    string method() const override { return "Cash"; }
};
class CardPayment : public Payment {
    double balance;
public:
    explicit CardPayment(double b) : balance(b) {}
    double fee(double a) const override { return a * 0.02; }
    void pay(double a) override {
        if (a > balance) throw PaymentException("Card balance is not enough (" + money(balance) + ")");
        balance -= a;
    }
    string method() const override { return "Credit card"; }
};
class EWalletPayment : public Payment {
    double balance;
public:
    explicit EWalletPayment(double b) : balance(b) {}
    double fee(double a) const override { return a * 0.01; }
    void pay(double a) override {
        if (a > balance) throw PaymentException("E-wallet balance is not enough (" + money(balance) + ")");
        balance -= a;
    }
    string method() const override { return "E-wallet"; }
};

class Restaurant {
    int id;
    string name;
    vector<shared_ptr<MenuItem>> menu;
public:
    Restaurant(int id, const string& name) : id(id), name(name) {}
    int getId() const { return id; }
    const string& getName() const { return name; }
    void addItem(shared_ptr<MenuItem> it) { menu.push_back(it); }
    size_t size() const { return menu.size(); }
    shared_ptr<MenuItem> findItem(int index) const {
        if (index < 1 || index > (int)menu.size()) throw NotFoundException("Menu item #" + to_string(index));
        return menu[index - 1];
    }
    shared_ptr<MenuItem> findItem(const string& keyword) const {
        for (auto& m : menu)
            if (m->getName().find(keyword) != string::npos) return m;
        throw NotFoundException("Menu item \"" + keyword + "\"");
    }
    void showMenu() const {
        cout << "  === " << name << " ===\n";
        for (size_t i = 0; i < menu.size(); ++i) cout << "   " << i + 1 << ". " << *menu[i] << "\n";
    }
};

enum class Status { PENDING, CONFIRMED, DELIVERING, DELIVERED, CANCELLED };
string statusStr(Status s) {
    switch (s) {
        case Status::PENDING: return "PENDING";
        case Status::CONFIRMED: return "CONFIRMED";
        case Status::DELIVERING: return "DELIVERING";
        case Status::DELIVERED: return "DELIVERED";
        default: return "CANCELLED";
    }
}

class Order {
    int id;
    shared_ptr<Customer> customer;
    shared_ptr<Restaurant> restaurant;
    shared_ptr<Driver> driver;
    vector<pair<shared_ptr<MenuItem>, int>> lines;
    Status status;
    string payMethod;
    double paidAmount;
public:
    Order(int id, shared_ptr<Customer> c, shared_ptr<Restaurant> r)
        : id(id), customer(c), restaurant(r), status(Status::PENDING), paidAmount(0) {}

    int getId() const { return id; }
    Status getStatus() const { return status; }

    void addItem(shared_ptr<MenuItem> it, int qty) {
        if (qty <= 0) throw InvalidInputException("Quantity must be > 0");
        lines.push_back({it, qty});
    }
    void addItem(shared_ptr<MenuItem> it) { addItem(it, 1); }
    Order& operator+=(shared_ptr<MenuItem> it) { addItem(it); return *this; }
    bool operator==(const Order& o) const { return id == o.id; }

    double subtotal() const {
        double s = 0;
        for (auto& l : lines) s += l.first->getPrice() * l.second;
        return s;
    }
    double total() const { return subtotal() * (1.0 - customer->discountRate()); }

    void checkout(Payment& p) {
        if (lines.empty()) throw OrderStateException("Order has no items");
        double t = total();
        double amount = t + p.fee(t);
        p.pay(amount);
        payMethod = p.method();
        paidAmount = amount;
        customer->addPoints(t);
    }

    void next(shared_ptr<Driver> d = nullptr) {
        switch (status) {
            case Status::PENDING: status = Status::CONFIRMED; break;
            case Status::CONFIRMED:
                if (!d) throw NotFoundException("No driver available");
                driver = d; driver->setAvailable(false);
                status = Status::DELIVERING; break;
            case Status::DELIVERING:
                driver->setAvailable(true);
                status = Status::DELIVERED; break;
            default:
                throw OrderStateException("Order #" + to_string(id) + " is already " + statusStr(status));
        }
    }
    void cancel() {
        if (status == Status::DELIVERING || status == Status::DELIVERED || status == Status::CANCELLED)
            throw OrderStateException("Cannot cancel an order that is " + statusStr(status));
        status = Status::CANCELLED;
    }
    void display() const {
        cout << "  Order #" << id << " | " << customer->getName() << " @ " << restaurant->getName()
             << " | " << statusStr(status) << "\n";
        for (auto& l : lines) cout << "     - " << l.second << " x " << l.first->describe() << "\n";
        cout << "     Subtotal: " << money(subtotal()) << " | Total after discount: " << money(total());
        if (!payMethod.empty()) cout << " | Paid " << money(paidAmount) << " by " << payMethod;
        if (driver) cout << " | Driver: " << driver->getName();
        cout << "\n";
    }
};

string readLine(const string& prompt) {
    cout << prompt;
    string s;
    if (!getline(cin, s)) { cout << "\nBye!\n"; exit(0); }
    return s;
}
int readInt(const string& prompt) {
    string s = readLine(prompt);
    try {
        size_t pos;
        int v = stoi(s, &pos);
        if (pos != s.size()) throw InvalidInputException("\"" + s + "\" is not a valid number");
        return v;
    } catch (const invalid_argument&) {
        throw InvalidInputException("\"" + s + "\" is not a valid number");
    } catch (const out_of_range&) {
        throw InvalidInputException("Number out of range");
    }
}

class DeliverySystem {
    vector<shared_ptr<Customer>> customers;
    vector<shared_ptr<Driver>> drivers;
    vector<shared_ptr<Restaurant>> restaurants;
    vector<shared_ptr<Order>> orders;
    Admin admin;
    int nextOrderId = 1, nextPersonId = 100;

    shared_ptr<Order> findOrder(int id) {
        for (auto& o : orders) if (o->getId() == id) return o;
        throw NotFoundException("Order #" + to_string(id));
    }
    shared_ptr<Driver> freeDriver() {
        for (auto& d : drivers) if (d->isAvailable()) return d;
        throw NotFoundException("No driver available right now");
    }
public:
    DeliverySystem() : admin(1, "System Admin", "0900000000") { seed(); }

    void seed() {
        customers.push_back(make_shared<Customer>(nextPersonId++, "Nguyen Van A", "0912345678", "Hai Ba Trung, Ha Noi"));
        customers.push_back(make_shared<VIPCustomer>(nextPersonId++, "Tran Thi B", "0987654321", "Cau Giay, Ha Noi"));
        auto dc = make_shared<DriverCustomer>(nextPersonId++, "Le Van C", "0933333333", "Dong Da, Ha Noi", "29A-123.45");
        customers.push_back(dc);
        drivers.push_back(dc);
        drivers.push_back(make_shared<Driver>(nextPersonId++, "Pham Van D", "0944444444", "30B-678.90"));

        auto r1 = make_shared<Restaurant>(1, "Pho 24");
        r1->addItem(make_shared<Food>("Pho Bo", 55000));
        r1->addItem(make_shared<Food>("Bun Cha", 50000, true));
        r1->addItem(make_shared<Drink>("Tra Da", 10000, 'L'));
        r1->addItem(make_shared<Combo>("Pho + Tra Da", vector<shared_ptr<MenuItem>>{
            make_shared<Food>("Pho Bo", 55000), make_shared<Drink>("Tra Da", 10000, 'M')}));
        auto r2 = make_shared<Restaurant>(2, "Pizza House");
        r2->addItem(make_shared<Food>("Pizza Hai San", 159000));
        r2->addItem(make_shared<Food>("Pizza Cay", 139000, true));
        r2->addItem(make_shared<Drink>("Coca Cola", 20000, 'M'));
        restaurants = {r1, r2};
    }

    void showRestaurants() const { for (auto& r : restaurants) r->showMenu(); }
    void showPeople() const {
        cout << "  --- Customers ---\n";
        for (auto& c : customers) c->display();
        cout << "  --- Drivers ---\n";
        for (auto& d : drivers) d->display();
        cout << "  --- Admin ---\n";
        admin.display();
        cout << "  Total Person objects alive: " << Person::getTotalPersons() << "\n";
    }

    void registerCustomer() {
        string name = readLine("  Name: "), phone = readLine("  Phone: "), addr = readLine("  Address: ");
        int vip = readInt("  VIP? (1 = yes, 0 = no): ");
        if (vip == 1) customers.push_back(make_shared<VIPCustomer>(nextPersonId++, name, phone, addr));
        else          customers.push_back(make_shared<Customer>(nextPersonId++, name, phone, addr));
        cout << "  Registered successfully.\n";
    }

    void createOrder() {
        for (size_t i = 0; i < customers.size(); ++i)
            cout << "   " << i + 1 << ". " << customers[i]->getName() << " (" << customers[i]->role() << ")\n";
        int ci = readInt("  Choose customer: ");
        if (ci < 1 || ci > (int)customers.size()) throw NotFoundException("Customer #" + to_string(ci));
        for (size_t i = 0; i < restaurants.size(); ++i)
            cout << "   " << i + 1 << ". " << restaurants[i]->getName() << "\n";
        int ri = readInt("  Choose restaurant: ");
        if (ri < 1 || ri > (int)restaurants.size()) throw NotFoundException("Restaurant #" + to_string(ri));

        auto rest = restaurants[ri - 1];
        rest->showMenu();
        auto order = make_shared<Order>(nextOrderId, customers[ci - 1], rest);
        while (true) {
            int item = readInt("  Item number (0 = finish): ");
            if (item == 0) break;
            try {
                auto it = rest->findItem(item);
                int qty = readInt("  Quantity: ");
                order->addItem(it, qty);
            } catch (const AppException& e) {
                cout << "  ! " << e.what() << "\n";
            }
        }
        cout << "  Payment: 1 = Cash, 2 = Card (balance 300,000), 3 = E-wallet (balance 1,000,000)\n";
        int pm = readInt("  Choose: ");
        unique_ptr<Payment> pay;
        if (pm == 1) pay = make_unique<CashPayment>();
        else if (pm == 2) pay = make_unique<CardPayment>(300000);
        else if (pm == 3) pay = make_unique<EWalletPayment>(1000000);
        else throw InvalidInputException("Unknown payment method");

        order->checkout(*pay);
        orders.push_back(order);
        ++nextOrderId;
        cout << "  Order created!\n";
        order->display();
    }

    void advanceOrder() {
        auto o = findOrder(readInt("  Order id: "));
        if (o->getStatus() == Status::CONFIRMED) o->next(freeDriver());
        else o->next();
        o->display();
    }
    void cancelOrder() {
        auto o = findOrder(readInt("  Order id: "));
        o->cancel();
        cout << "  Order cancelled.\n";
    }
    void showOrders() const {
        if (orders.empty()) cout << "  (no orders)\n";
        for (auto& o : orders) o->display();
    }

    void demo() {
        cout << "\n  --- DEMO 1: Polymorphism via vector<shared_ptr<Person>> ---\n";
        vector<shared_ptr<Person>> all;
        for (auto& c : customers) all.push_back(c);
        for (auto& d : drivers) if (d->getId() != customers[2]->getId()) all.push_back(d);
        all.push_back(make_shared<Admin>(2, "Boss", "0911111111"));
        for (auto& p : all) p->display();

        cout << "\n  --- DEMO 2: Polymorphic payment fee on 100,000 VND ---\n";
        vector<unique_ptr<Payment>> pays;
        pays.push_back(make_unique<CashPayment>());
        pays.push_back(make_unique<CardPayment>(1e6));
        pays.push_back(make_unique<EWalletPayment>(1e6));
        for (auto& p : pays) cout << "   " << p->method() << " fee: " << money(p->fee(100000)) << "\n";

        cout << "\n  --- DEMO 3: Diamond problem solved by virtual inheritance ---\n";
        DriverCustomer dc(500, "Demo", "0955555555", "Somewhere", "11A-111.11");
        Customer* c = &dc; Driver* d = &dc;
        cout << "   Same Person subobject? " << (static_cast<Person*>(c) == static_cast<Person*>(d) ? "yes" : "no") << "\n";
        cout << "   Name via Customer*: " << c->getName() << " | via Driver*: " << d->getName() << "\n";

        cout << "\n  --- DEMO 4: Exceptions ---\n";
        try { Customer bad(1, "", "0912345678", "x"); }
        catch (const InvalidInputException& e) { cout << "   caught: " << e.what() << "\n"; }
        try { restaurants[0]->findItem("Sushi"); }
        catch (const NotFoundException& e) { cout << "   caught: " << e.what() << "\n"; }
        try { restaurants[0]->findItem(99); }
        catch (const exception& e) { cout << "   caught (base std::exception): " << e.what() << "\n"; }
        try { Order o(999, customers[0], restaurants[0]); o.addItem(restaurants[0]->findItem(1), 0); }
        catch (const AppException& e) { cout << "   caught: " << e.what() << "\n"; }
        try { Order o(999, customers[0], restaurants[0]); CashPayment cash; o.checkout(cash); }
        catch (const OrderStateException& e) { cout << "   caught: " << e.what() << "\n"; }
        try { CardPayment card(1000); card.pay(5000); }
        catch (const PaymentException& e) { cout << "   caught: " << e.what() << "\n"; }
        catch (...) { cout << "   unknown error\n"; }

        cout << "\n  --- DEMO 5: Operator overloading ---\n";
        Order o1(1, customers[1], restaurants[0]), o2(1, customers[0], restaurants[1]);
        o1 += restaurants[0]->findItem(1);
        o1 += restaurants[0]->findItem("Bun");
        cout << "   o1 == o2 (same id)? " << (o1 == o2 ? "yes" : "no") << "\n";
        cout << "   o1 subtotal " << money(o1.subtotal()) << " -> VIP total " << money(o1.total()) << "\n";
    }
};

int main() {
    DeliverySystem sys;
    while (true) {
        cout << "\n========== FOOD DELIVERY MANAGEMENT SYSTEM ==========\n"
             << " 1. Show restaurants & menus\n 2. Show people (customers/drivers/admin)\n"
             << " 3. Register customer\n 4. Create order\n 5. Advance order status\n"
             << " 6. Cancel order\n 7. Show orders\n 8. Demo (inheritance/polymorphism/exceptions)\n 0. Exit\n";
        try {
            int ch = readInt("Choose: ");
            switch (ch) {
                case 1: sys.showRestaurants(); break;
                case 2: sys.showPeople(); break;
                case 3: sys.registerCustomer(); break;
                case 4: sys.createOrder(); break;
                case 5: sys.advanceOrder(); break;
                case 6: sys.cancelOrder(); break;
                case 7: sys.showOrders(); break;
                case 8: sys.demo(); break;
                case 0: cout << "Goodbye!\n"; return 0;
                default: throw InvalidInputException("Menu choice must be 0-8");
            }
        } catch (const AppException& e) {
            cout << "  ERROR: " << e.what() << "\n";
        } catch (const exception& e) {
            cout << "  SYSTEM ERROR: " << e.what() << "\n";
        } catch (...) {
            cout << "  UNKNOWN ERROR\n";
        }
    }
}
