# EV-Charging-Management-System
The EV Charging Management System is a C++ application designed to manage electric vehicle charging operations. It helps users manage charging sessions, calculate charging costs, and maintain charging records.The project demonstrates the practical application of object-oriented programming concepts in C++.


#include <iostream>
#include <iomanip>
#include <string>
#include <vector>
#include <fstream>
#include <sstream>
#include <algorithm>
#include <ctime>
#include <cctype>

using namespace std;

const string ACTIVE_FILE = "ev_active_sessions.txt";
const string HISTORY_FILE = "ev_charging_history.txt";

// ================= GENERAL FUNCTIONS =================

void line(char c = '=', int n = 76) {
    cout << string(n, c) << '\n';
}

void title(const string& text) {
    cout << '\n';
    line();
    cout << "  " << text << '\n';
    line();
}

string readLine(const string& prompt) {
    string value;
    cout << prompt;
    getline(cin, value);
    return value;
}

int readInt(const string& prompt, int low, int high) {
    while (true) {
        cout << prompt;
        string s;
        getline(cin, s);

        stringstream ss(s);
        int value;
        char extra;

        if (ss >> value && !(ss >> extra) &&
            value >= low && value <= high)
            return value;

        cout << "Invalid input. Enter a number from "
             << low << " to " << high << ".\n";
    }
}

double readDouble(const string& prompt, double low, double high) {
    while (true) {
        cout << prompt;
        string s;
        getline(cin, s);

        stringstream ss(s);
        double value;
        char extra;

        if (ss >> value && !(ss >> extra) &&
            value >= low && value <= high)
            return value;

        cout << "Invalid value. Please try again.\n";
    }
}

string readName(const string& prompt) {
    while (true) {
        string name;
        cout << prompt;
        getline(cin, name);

        bool valid = !name.empty();
        bool hasLetter = false;

        for (char c : name) {
            if (isalpha(static_cast<unsigned char>(c))) {
                hasLetter = true;
            } else if (c != ' ') {
                valid = false;
                break;
            }
        }

        if (valid && hasLetter)
            return name;

        cout << "Invalid name! Use letters and spaces only.\n";
    }
}

string readPhone(const string& prompt) {
    while (true) {
        string phone;
        cout << prompt;
        getline(cin, phone);

        bool valid = phone.size() == 10;

        for (char c : phone) {
            if (!isdigit(static_cast<unsigned char>(c))) {
                valid = false;
                break;
            }
        }

        if (valid)
            return phone;

        cout << "Invalid mobile number! Enter exactly 10 digits.\n";
    }
}

string currentTime() {
    time_t now = time(nullptr);
    tm* local = std::localtime(&now);

    if (!local)
        return "Time unavailable";

    char buffer[32];
    strftime(buffer, sizeof(buffer),
             "%d-%m-%Y %H:%M:%S", local);

    return buffer;
}

string money(double amount) {
    ostringstream out;
    out << "Rs. " << fixed << setprecision(2) << amount;
    return out.str();
}

vector<string> splitRecord(const string& row) {
    vector<string> fields;
    string part;
    stringstream ss(row);

    while (getline(ss, part, '|'))
        fields.push_back(part);

    return fields;
}

// ================= CHARGING SESSION CLASS =================

class ChargingSession {
public:
    int sessionId = 0;

    string customerName;
    string phone;
    string registration;
    string vehicleType;
    string chargingMode;
    string startTime;

    double batteryCapacity = 0;
    double startPercentage = 0;
    double targetPercentage = 0;
    double energyRequired = 0;
    double chargerPower = 0;
    double ratePerUnit = 0;
    double chargingHours = 0;
    double energyCost = 0;
    double serviceFee = 0;
    double tax = 0;
    double totalCost = 0;
};

// ================= EV CHARGING SYSTEM =================

class EVChargingSystem {
private:
    vector<ChargingSession> activeSessions;
    int nextSessionId = 1001;

    // Rates are sample values in INR per kWh.
    // Charger power is in kW.
    struct ChargingOption {
        string name;
        double power;
        double rate;
    };

    vector<ChargingOption> options = {
        {"AC Slow",       3.3,  10.0},
        {"AC Standard",   7.4,  14.0},
        {"AC Fast",      11.0,  18.0},
        {"DC Fast",      30.0,  22.0},
        {"DC Ultra-Fast",60.0,  28.0}
    };

    ChargingOption selectedOption(int choice) {
        return options.at(choice - 1);
    }

    // Recover unique IDs from active and completed records.
    void loadNextSessionId() {
        int maxId = 1000;
        string row;

        ifstream history(HISTORY_FILE);

        while (getline(history, row)) {
            vector<string> f = splitRecord(row);

            if (!f.empty()) {
                try {
                    maxId = max(maxId, stoi(f[0]));
                } catch (...) {}
            }
        }

        ifstream active(ACTIVE_FILE);

        while (getline(active, row)) {
            vector<string> f = splitRecord(row);

            if (!f.empty()) {
                try {
                    maxId = max(maxId, stoi(f[0]));
                } catch (...) {}
            }
        }

        nextSessionId = maxId + 1;
    }

    void loadActiveSessions() {
        ifstream in(ACTIVE_FILE);
        string row;

        while (getline(in, row)) {
            vector<string> f = splitRecord(row);

            if (f.size() != 15)
                continue;

            try {
                ChargingSession s;

                s.sessionId = stoi(f[0]);
                s.customerName = f[1];
                s.phone = f[2];
                s.registration = f[3];
                s.vehicleType = f[4];
                s.chargingMode = f[5];
                s.startTime = f[6];
                s.batteryCapacity = stod(f[7]);
                s.startPercentage = stod(f[8]);
                s.targetPercentage = stod(f[9]);
                s.energyRequired = stod(f[10]);
                s.chargerPower = stod(f[11]);
                s.ratePerUnit = stod(f[12]);
                s.chargingHours = stod(f[13]);
                s.energyCost = stod(f[14]);

                s.serviceFee = s.energyCost * 0.05;
                s.tax = (s.energyCost + s.serviceFee) * 0.18;
                s.totalCost = s.energyCost + s.serviceFee + s.tax;

                bool duplicate = false;

                for (const auto& existing : activeSessions) {
                    if (existing.sessionId == s.sessionId) {
                        duplicate = true;
                        break;
                    }
                }

                if (!duplicate)
                    activeSessions.push_back(s);

            } catch (...) {
                cout << "Warning: Invalid session record skipped.\n";
            }
        }
    }

    bool saveActiveSessions() {
        ofstream out(ACTIVE_FILE);

        if (!out)
            return false;

        for (const auto& s : activeSessions) {
            out << s.sessionId << '|'
                << s.customerName << '|'
                << s.phone << '|'
                << s.registration << '|'
                << s.vehicleType << '|'
                << s.chargingMode << '|'
                << s.startTime << '|'
                << s.batteryCapacity << '|'
                << s.startPercentage << '|'
                << s.targetPercentage << '|'
                << s.energyRequired << '|'
                << s.chargerPower << '|'
                << s.ratePerUnit << '|'
                << s.chargingHours << '|'
                << s.energyCost << '\n';
        }

        return static_cast<bool>(out);
    }

    void showChargingOptions() {
        title("CHARGING MODE AND RATES");

        cout << left
             << setw(6)  << "No."
             << setw(19) << "Mode"
             << setw(16) << "Power (kW)"
             << "Rate (Rs./kWh)\n";

        line('-');

        for (size_t i = 0; i < options.size(); ++i) {
            cout << left
                 << setw(6) << i + 1
                 << setw(19) << options[i].name
                 << setw(16) << fixed << setprecision(1)
                 << options[i].power
                 << fixed << setprecision(2)
                 << options[i].rate << '\n';
        }
    }

    void createInvoice(const ChargingSession& s) {
        string filename = "EV_Invoice_" +
                          to_string(s.sessionId) + ".txt";

        ofstream out(filename);

        if (!out) {
            cout << "Error creating invoice file.\n";
            return;
        }

        out << "========================================\n";
        out << "       EV CHARGING MANAGEMENT\n";
        out << "                INVOICE\n";
        out << "========================================\n\n";

        out << "Session ID          : " << s.sessionId << '\n';
        out << "Date and Time       : " << s.startTime << '\n';
        out << "Customer            : " << s.customerName << '\n';
        out << "Mobile Number       : " << s.phone << '\n';
        out << "Vehicle Registration: " << s.registration << '\n';
        out << "Vehicle Type        : " << s.vehicleType << '\n';

        out << "\n----------------------------------------\n";
        out << "Charging Mode       : " << s.chargingMode << '\n';
        out << "Battery Capacity    : "
            << s.batteryCapacity << " kWh\n";
        out << "Starting Battery    : "
            << s.startPercentage << "%\n";
        out << "Target Battery      : "
            << s.targetPercentage << "%\n";
        out << "Energy Required     : "
            << fixed << setprecision(2)
            << s.energyRequired << " kWh\n";
        out << "Charger Power       : "
            << s.chargerPower << " kW\n";
        out << "Estimated Duration  : "
            << s.chargingHours << " hours\n";

        out << "\n----------------------------------------\n";
        out << "Energy Charges      : " << money(s.energyCost) << '\n';
        out << "Service Fee (5%)    : " << money(s.serviceFee) << '\n';
        out << "GST (18%)           : " << money(s.tax) << '\n';
        out << "TOTAL CHARGES       : " << money(s.totalCost) << '\n';

        out << "\nNote: This is a simulated charging estimate.\n";
        out << "========================================\n";
        out.close();

        if (!out) {
            cout << "Error writing invoice file.\n";
            return;
        }

        ifstream in(filename);
        string row;

        while (getline(in, row))
            cout << row << '\n';

        cout << "\nInvoice saved: " << filename << '\n';
    }

    void startSession() {
        title("START EV CHARGING SESSION");

        showChargingOptions();

        ChargingSession s;
        s.sessionId = nextSessionId;

        s.customerName = readName("\nCustomer name: ");
        s.phone = readPhone("Mobile number (10 digits): ");

        s.registration = readLine("Vehicle registration number: ");

        if (s.registration.empty() ||
            s.registration.find('|') != string::npos) {
            cout << "Invalid registration number.\n";
            return;
        }

        cout << "\nVehicle types:\n"
             << "1. Electric Scooter\n"
             << "2. Electric Motorcycle\n"
             << "3. Electric Car\n"
             << "4. Electric SUV\n"
             << "5. Electric Bus / Commercial\n";

        int type = readInt("Select vehicle type: ", 1, 5);

        string types[] = {
            "", "Electric Scooter", "Electric Motorcycle",
            "Electric Car", "Electric SUV",
            "Electric Bus / Commercial"
        };

        s.vehicleType = types[type];

        s.batteryCapacity = readDouble(
            "Battery capacity (kWh): ", 0.1, 1000);

        s.startPercentage = readDouble(
            "Current battery percentage (0-100): ", 0, 100);

        s.targetPercentage = readDouble(
            "Target battery percentage (1-100): ", 1, 100);

        if (s.targetPercentage <= s.startPercentage) {
            cout << "Target battery must be greater than current battery.\n";
            return;
        }

        int mode = readInt(
            "\nSelect charging mode (1-5): ", 1, 5);

        ChargingOption option = selectedOption(mode);

        s.chargingMode = option.name;
        s.chargerPower = option.power;
        s.ratePerUnit = option.rate;

        // Estimate energy needed. This is a simplified model.
        s.energyRequired =
            s.batteryCapacity *
            (s.targetPercentage - s.startPercentage) / 100.0;

        // Estimated duration: energy / charger power.
        s.chargingHours = s.energyRequired / s.chargerPower;

        // Approximate 10% charging losses.
        s.energyRequired *= 1.10;
        s.chargingHours = s.energyRequired / s.chargerPower;

        s.energyCost = s.energyRequired * s.ratePerUnit;
        s.serviceFee = s.energyCost * 0.05;
        s.tax = (s.energyCost + s.serviceFee) * 0.18;
        s.totalCost = s.energyCost + s.serviceFee + s.tax;

        s.startTime = currentTime();

        activeSessions.push_back(s);

        if (!saveActiveSessions()) {
            activeSessions.pop_back();
            cout << "Could not save session. Please try again.\n";
            return;
        }

        ++nextSessionId;

        // Generate the invoice immediately.
        createInvoice(s);

        cout << "\nCharging session started successfully.\n";
        cout << "Estimated energy: "
             << fixed << setprecision(2)
             << s.energyRequired << " kWh\n";
        cout << "Estimated duration: "
             << s.chargingHours << " hours\n";
        cout << "Estimated total: " << money(s.totalCost) << '\n';
    }

    void showActiveSessions() {
        title("ACTIVE CHARGING SESSIONS");

        if (activeSessions.empty()) {
            cout << "No active charging sessions.\n";
            return;
        }

        for (const auto& s : activeSessions) {
            cout << "\nSession ID: " << s.sessionId
                 << "\nCustomer: " << s.customerName
                 << "\nVehicle: " << s.registration
                 << "\nCharging Mode: " << s.chargingMode
                 << "\nEnergy Required: "
                 << fixed << setprecision(2)
                 << s.energyRequired << " kWh"
                 << "\nEstimated Cost: " << money(s.totalCost)
                 << '\n';

            line('-');
        }
    }

    void completeSession() {
        title("COMPLETE CHARGING SESSION");

        if (activeSessions.empty()) {
            cout << "No active sessions to complete.\n";
            return;
        }

        showActiveSessions();

        int id = readInt(
            "Enter session ID to complete (0 to cancel): ",
            0, 999999);

        if (id == 0)
            return;

        auto it = find_if(activeSessions.begin(), activeSessions.end(),
            [id](const ChargingSession& s) {
                return s.sessionId == id;
            });

        if (it == activeSessions.end()) {
            cout << "Session ID not found.\n";
            return;
        }

        ChargingSession s = *it;

        // Append completed session to history.
        ofstream history(HISTORY_FILE, ios::app);

        if (!history) {
            cout << "Could not save charging history.\n";
            return;
        }

        history << s.sessionId << '|'
                << s.customerName << '|'
                << s.phone << '|'
                << s.registration << '|'
                << s.vehicleType << '|'
                << s.chargingMode << '|'
                << s.energyRequired << '|'
                << s.chargingHours << '|'
                << s.energyCost << '|'
                << s.serviceFee << '|'
                << s.tax << '|'
                << s.totalCost << '|'
                << currentTime() << '\n';

        history.close();

        if (!history) {
            cout << "Could not write charging history.\n";
            return;
        }

        string filename = "EV_Complete_" +
                          to_string(s.sessionId) + ".txt";

        ofstream out(filename);

        if (out) {
            out << "========================================\n";
            out << "       EV CHARGING COMPLETION\n";
            out << "========================================\n\n";
            out << "Session ID       : " << s.sessionId << '\n';
            out << "Customer         : " << s.customerName << '\n';
            out << "Vehicle          : " << s.registration << '\n';
            out << "Charging Mode    : " << s.chargingMode << '\n';
            out << "Energy Delivered : "
                << fixed << setprecision(2)
                << s.energyRequired << " kWh\n";
            out << "Estimated Time   : "
                << s.chargingHours << " hours\n";
            out << "Energy Charges   : " << money(s.energyCost) << '\n';
            out << "Service Fee      : " << money(s.serviceFee) << '\n';
            out << "GST              : " << money(s.tax) << '\n';
            out << "TOTAL PAID       : " << money(s.totalCost) << '\n';
            out << "Completed At     : " << currentTime() << '\n';
            out << "========================================\n";
        }

        activeSessions.erase(it);

        if (!saveActiveSessions())
            cout << "Warning: Could not update active sessions file.\n";

        cout << "\nSession completed successfully.\n";
        cout << "Completion record saved: " << filename << '\n';
    }

    void showHistory() {
        title("CHARGING HISTORY & REVENUE REPORT");

        ifstream in(HISTORY_FILE);

        if (!in) {
            cout << "No charging history found yet.\n";
            return;
        }

        string row;
        int count = 0;
        double totalEnergy = 0;
        double totalRevenue = 0;

        while (getline(in, row)) {
            vector<string> f = splitRecord(row);

            if (f.size() != 13)
                continue;

            try {
                int id = stoi(f[0]);
                double energy = stod(f[6]);
                double total = stod(f[11]);

                ++count;
                totalEnergy += energy;
                totalRevenue += total;

                cout << "Session #" << id
                     << " | " << f[1]
                     << " | " << f[3]
                     << " | " << f[5]
                     << " | Energy: " << fixed << setprecision(2)
                     << energy << " kWh"
                     << " | Total: " << money(total)
                     << '\n';
            } catch (...) {
                cout << "Warning: Invalid history record skipped.\n";
            }
        }

        line('-');

        cout << "Completed sessions : " << count << '\n';
        cout << "Total energy       : "
             << fixed << setprecision(2)
             << totalEnergy << " kWh\n";
        cout << "Total revenue      : " << money(totalRevenue) << '\n';
        cout << "Active sessions    : " << activeSessions.size() << '\n';
    }

public:
    EVChargingSystem() {
        loadVehiclesPlaceholder();
        loadActiveSessions();
        loadNextSessionId();
    }

    void loadVehiclesPlaceholder() {
        // This system models charging sessions rather than physical bays.
    }

    void run() {
        while (true) {
            title("EV CHARGING MANAGEMENT SYSTEM");

            cout << "1. Start Charging Session (Instant Invoice)\n"
                 << "2. View Active Sessions\n"
                 << "3. Complete Charging Session\n"
                 << "4. Charging History & Revenue Report\n"
                 << "0. Exit\n";

            int choice = readInt("\nEnter your choice: ", 0, 4);

            switch (choice) {
                case 1:
                    startSession();
                    break;

                case 2:
                    showActiveSessions();
                    break;

                case 3:
                    completeSession();
                    break;

                case 4:
                    showHistory();
                    break;

                case 0:
                    cout << "\nThank you for using EV Charging Management System!\n";
                    return;
            }

            cout << "\nPress Enter to continue...";
            string dummy;
            getline(cin, dummy);
        }
    }
};

int main() {
    EVChargingSystem system;
    system.run();
    return 0;
}