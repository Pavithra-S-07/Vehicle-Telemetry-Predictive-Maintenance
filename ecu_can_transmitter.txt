#include <Arduino.h>

#include <SPI.h>
#include <mcp2515.h>

struct can_frame canMsg;
struct can_frame ackMsg;

MCP2515 mcp2515(5); // CS pin

#define MAX_RETRIES 3
#define ENGINE_CAN_ID 0x036
#define CAN_ACK_ID    0x037

// ----------------------
// Realistic engine state
// ----------------------
float rpm = 900;
float speed = 0;
float coolant = 80;
float throttle = 10;

float rpmTarget = 900;
float throttleTarget = 10;

void setup()
{
    Serial.begin(115200);
    SPI.begin();

    mcp2515.reset();
    mcp2515.setBitrate(CAN_500KBPS, MCP_8MHZ);
    mcp2515.setNormalMode();

    Serial.println("Fake ECU Transmitter Started (Realistic Mode)");
}

void loop()
{
    // -----------------------------
    // 1. Simulate driver throttle
    // -----------------------------
    throttleTarget += random(-3, 4);   // slow driver variation
    throttleTarget = constrain(throttleTarget, 0, 100);

    // smooth throttle movement (driver pedal feel)
    throttle += (throttleTarget - throttle) * 0.1;

    // -----------------------------
    // 2. Engine RPM model
    // -----------------------------
    rpmTarget = 800 + (throttle * 40);  // idle + load effect

    // inertia smoothing (engine response delay)
    rpm += (rpmTarget - rpm) * 0.15;

    // -----------------------------
    // 3. Vehicle speed model
    // -----------------------------
    float speedTarget = (rpm - 800) / 35.0;

    if (speedTarget < 0) speedTarget = 0;

    // simulate drivetrain lag
    speed += (speedTarget - speed) * 0.1;

    // -----------------------------
    // 4. Coolant temperature model
    // -----------------------------
    float coolantTarget = 75 + (throttle * 0.25);

    // slow thermal response
    coolant += (coolantTarget - coolant) * 0.03;

    // -----------------------------
    // 5. Clamp values (safe ranges)
    // -----------------------------
    rpm = constrain(rpm, 800, 5000);
    speed = constrain(speed, 0, 180);
    coolant = constrain(coolant, 70, 115);
    throttle = constrain(throttle, 0, 100);

    // -----------------------------
    // 6. Prepare CAN packet
    // -----------------------------
    canMsg.can_id = ENGINE_CAN_ID;
    canMsg.can_dlc = 5;

    canMsg.data[0] = ((uint16_t)rpm >> 8) & 0xFF;
    canMsg.data[1] = (uint16_t)rpm & 0xFF;
    canMsg.data[2] = (uint8_t)speed;
    canMsg.data[3] = (uint8_t)coolant;
    canMsg.data[4] = (uint8_t)throttle;

    // -----------------------------
    // 7. Send with retry + ACK
    // -----------------------------
    bool messageSent = false;
    int retries = 0;

    while (!messageSent && retries < MAX_RETRIES)
    {
        if (mcp2515.sendMessage(&canMsg) == MCP2515::ERROR_OK)
        {
            Serial.println("===== ENGINE DATA SENT =====");

            Serial.print("RPM: ");
            Serial.println((int)rpm);

            Serial.print("Speed: ");
            Serial.println((int)speed);

            Serial.print("Coolant: ");
            Serial.println((int)coolant);

            Serial.print("Throttle: ");
            Serial.println((int)throttle);

            // Wait for ACK
            unsigned long startTime = millis();
            bool ackReceived = false;

            while (millis() - startTime < 300)
            {
                if (mcp2515.readMessage(&ackMsg) == MCP2515::ERROR_OK)
                {
                    if (ackMsg.can_id == CAN_ACK_ID)
                    {
                        ackReceived = true;
                        break;
                    }
                }
            }

            if (ackReceived)
            {
                Serial.println("ACK RECEIVED");
                messageSent = true;
            }
            else
            {
                Serial.println("ACK NOT RECEIVED -> RETRY");
                retries++;
            }
        }
        else
        {
            Serial.println("CAN SEND ERROR");
            retries++;
        }
    }

    if (!messageSent)
    {
        Serial.println("FAILED AFTER MAX RETRIES");
    }

    Serial.println("-----------------------------");
    delay(200);
}