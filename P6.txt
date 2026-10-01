#include <Arduino.h>

/// Wiring
// PD2 -> EN  (Motor)
// PD3 -> DIR (Direction)
// PD4 -> PUL (Step pulse)

// ! Do not set M012 = 0 0 0

#define EN_PIN   2
#define DIR_PIN  3
#define PUL_PIN  4

/// Motor configuration
// 400 pulses = 360 degrees
// M012 = 1 0 0 -> Half Step
const float STEPS_PER_DEGREE = 400.0 / 360.0;

// X List of angles to test
const uint16_t target_angles[] = {
    0, 45, 90, 135, 180, 225, 270, 315, 360
};

// X Calculate number of angles
const uint8_t TOTAL_ANGLES =
    sizeof(target_angles) / sizeof(target_angles[0]);

// Current motor position
uint16_t current_angle = 0;


// Generate step pulses
void stepMotor(uint16_t steps, bool direction)
{
    // Set rotation direction
    digitalWrite(DIR_PIN, direction ? HIGH : LOW);

    // Generate the required number of pulses
    for (uint16_t i = 0; i < steps; i++)
    {
        digitalWrite(PUL_PIN, HIGH);
        delayMicroseconds(1000);

        digitalWrite(PUL_PIN, LOW);
        delayMicroseconds(1000);
    }
}


// Move motor to target angle
void moveToAngle(uint16_t target_angle)
{
    // Difference between target and current position
    int16_t angle_diff = target_angle - current_angle;

    // No movement needed
    if (angle_diff == 0)
        return;

    // Positive -> one direction
    // Negative -> opposite direction
    bool direction = (angle_diff > 0);

    // Get absolute angle difference
    uint16_t abs_angle_diff = abs(angle_diff);

    // Convert angle to number of steps
    uint16_t steps =
        round(abs_angle_diff * STEPS_PER_DEGREE);

    // Move motor
    stepMotor(steps, direction);

    // Remember new position
    current_angle = target_angle;
}


/// Setup
void setup()
{
    pinMode(EN_PIN, OUTPUT);
    pinMode(DIR_PIN, OUTPUT);
    pinMode(PUL_PIN, OUTPUT);

    // DRV8825 Enable is active LOW
    digitalWrite(EN_PIN, LOW);
}

/// Main program
void loop()
{
    // X Demo:
    // Move through several predefined angles
    for (uint8_t i = 0; i < TOTAL_ANGLES; i++)
    {
        moveToAngle(target_angles[i]);

        // X Demo: wait at each position
        delay(1000);
    }

    // X Demo:
    // The next cycle starts from 0 degrees.
    // The motor physically returns to 0 through the
    // first command in target_angles[].
    current_angle = 0;
}