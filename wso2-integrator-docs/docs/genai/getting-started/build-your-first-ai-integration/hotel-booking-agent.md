---
title: "Build a Sample Hotel Booking Agent"
description: Build a hotel booking agent step by step with search, availability, and booking tools in WSO2 Integrator.
---

# Build a Sample Hotel Booking Agent

In this tutorial, you will build a **Hotel Booking Assistant** chat agent that can search for hotels, check room availability, and make reservations. By the end, you will have a working conversational agent that handles multi-turn booking workflows using tools and session memory.

This tutorial follows the same visual-designer-first approach as the [Smart Calculator Assistant](smart-calculator-assistant.md) tutorial. If you are already comfortable with that workflow, this tutorial adds more realistic tool logic and multi-step user interactions.

## What You'll Build

A chat agent with three capabilities:

- **Search hotels** -- Find hotels by city, check-in date, and number of guests
- **Check availability** -- Verify room availability and pricing for a specific hotel
- **Make a booking** -- Reserve a room and return a confirmation number

## Prerequisites

- [WSO2 Integrator VS Code extension installed](/docs/get-started/install)
- BI Copilot sign-in configured (for the default WSO2 model provider)
- Familiarity with the [Smart Calculator Assistant](smart-calculator-assistant.md) tutorial

## Step 1: Create a new integration project

1. Click the **WSO2 Integrator: BI** icon in the sidebar.
2. Click the **Create New Integration** button.
3. Enter the project name as `HotelBookingAgent`.
4. Select the project directory location and click **Create New Integration**.

## Step 2: Create an agent

1. Click the **+** button on the WSO2 Integrator side panel.
2. Select **AI Chat Agent** under the **AI Agent** artifacts.
3. Name the agent `BookingAssistant`.
4. Wait for the agent to be created with default configuration.

## Step 3: Configure the agent behavior

Click the **AI Agent** box and configure:

**Role:**

```text
Hotel Booking Assistant
```

**Instructions:**

```text
You are a hotel booking assistant. Help users find hotels, check room availability, and make reservations.

Your responsibilities:
- Search for hotels in a given city based on check-in date and number of guests.
- Check availability and pricing for a specific hotel.
- Make bookings when the user confirms their choice.

Guidelines:
- Always confirm the city, dates, and number of guests before searching.
- Present search results in a clear, readable format with pricing.
- Before making a booking, summarize the selection and ask for confirmation.
- If no hotels are available, suggest alternative dates or nearby cities.
```

Click **Save** to apply.

:::tip
Booking workflows involve multiple steps. Write instructions that tell the agent to confirm details before taking action -- this prevents unintended bookings.
:::

## Step 4: Configure the agent model

By default, the agent uses the **Default Model Provider (WSO2)**. Ensure you are signed in to BI Copilot:

- Press `Ctrl/Cmd` + `Shift` + `P` to open the VS Code Command Palette.
- Run the command: **`Ballerina: Configure default WSO2 model provider`**.

To use a different provider (e.g., OpenAI), follow the model configuration steps in the [Smart Calculator Assistant](smart-calculator-assistant.md#step-4-configure-the-agent-model) tutorial.

## Step 5: Create the tool functions

Create three functions that the agent will use as tools. For each function, click the **+** button in the WSO2 Integrator side panel under the **Functions** section.

### Pro-code implementation

Here is the complete pro-code implementation of the agent with all three tools:

```ballerina
import ballerina/ai;
import ballerina/http;
import ballerina/time;
import ballerina/uuid;

// --- Types ---

type Hotel record {|
    string id;
    string name;
    string city;
    decimal pricePerNight;
    int availableRooms;
    string roomType;
|};

type BookingConfirmation record {|
    string confirmationId;
    string hotelName;
    string city;
    string checkIn;
    string checkOut;
    int guests;
    decimal totalPrice;
|};

// --- In-memory data store ---

isolated map<Hotel> hotels = {
    "h1": {id: "h1", name: "Grand Plaza", city: "Colombo", pricePerNight: 150.00, availableRooms: 5, roomType: "Deluxe"},
    "h2": {id: "h2", name: "Ocean View Resort", city: "Colombo", pricePerNight: 220.00, availableRooms: 3, roomType: "Suite"},
    "h3": {id: "h3", name: "Mountain Lodge", city: "Kandy", pricePerNight: 95.00, availableRooms: 8, roomType: "Standard"},
    "h4": {id: "h4", name: "Lakeside Inn", city: "Kandy", pricePerNight: 130.00, availableRooms: 4, roomType: "Deluxe"},
    "h5": {id: "h5", name: "Beach House", city: "Galle", pricePerNight: 180.00, availableRooms: 6, roomType: "Suite"}
};

isolated map<BookingConfirmation> bookings = {};

// --- Tool functions ---

@ai:AgentTool
isolated function searchHotels(string city, int guests) returns Hotel[] {
    lock {
        return hotels.toArray()
            .filter(h => h.city.equalsIgnoreCaseAscii(city) && h.availableRooms >= guests)
            .clone();
    }
}

@ai:AgentTool
isolated function checkAvailability(string hotelId) returns Hotel|string {
    lock {
        Hotel? hotel = hotels[hotelId];
        if hotel is () {
            return string `No hotel found with ID '${hotelId}'.`;
        }
        return hotel.clone();
    }
}

@ai:AgentTool
isolated function makeBooking(string hotelId, string checkIn, string checkOut, int guests) returns BookingConfirmation|string {
    lock {
        Hotel? hotel = hotels[hotelId];
        if hotel is () {
            return string `No hotel found with ID '${hotelId}'.`;
        }
        if hotel.availableRooms < 1 {
            return string `No rooms available at ${hotel.name}.`;
        }

        string confirmationId = "BK-" + uuid:createRandomUuid().substring(0, 8).toUpperAscii();
        BookingConfirmation confirmation = {
            confirmationId: confirmationId,
            hotelName: hotel.name,
            city: hotel.city,
            checkIn: checkIn,
            checkOut: checkOut,
            guests: guests,
            totalPrice: hotel.pricePerNight * 2 // simplified: 2-night stay
        };
        bookings[confirmationId] = confirmation.clone();

        // Decrement available rooms
        hotels[hotelId] = {
            ...hotel,
            availableRooms: hotel.availableRooms - 1
        };

        return confirmation.clone();
    }
}

// --- Agent ---

final ai:Agent bookingAgent = check new ({
    systemPrompt: {
        role: "Hotel Booking Assistant",
        instructions: string `You are a hotel booking assistant. Help users find
            hotels, check room availability, and make reservations.
            Always confirm details before making a booking.
            Present hotel results clearly with pricing and room type.`
    },
    tools: [searchHotels, checkAvailability, makeBooking],
    model: check ai:getDefaultModelProvider()
});

// --- Chat service ---

service /booking on new ai:Listener(8080) {
    resource function post chat(@http:Payload ai:ChatReqMessage request)
                        returns ai:ChatRespMessage|error {
        string response = check bookingAgent.run(request.message, request.sessionId);
        return {message: response};
    }
}
```

### Adding tools via the visual designer

If you prefer the visual designer:

1. Create each function (`searchHotels`, `checkAvailability`, `makeBooking`) using the **Functions** section.
2. Go to the agent flow view.
3. Click the **+** button at the bottom-right of the `AI Agent` box.
4. Select **Use Function** and pick each function from the list.
5. Provide descriptive tool names and descriptions for each.

:::note
The more descriptive your tool descriptions, the better the LLM will be at choosing the right tool. For example, describe `searchHotels` as "Search for available hotels in a given city for a specified number of guests" rather than just "Search hotels".
:::

## Step 6: Run and test

1. Click the **Chat** button at the top-left corner of the interface.
2. Click **Run Integration**.
3. Start a conversation with your booking assistant:

```
You: I want to find a hotel in Colombo for 2 guests.
Agent: I found 2 hotels in Colombo:
  1. Grand Plaza - Deluxe room, $150/night, 5 rooms available
  2. Ocean View Resort - Suite, $220/night, 3 rooms available
  Which one interests you?

You: I'd like to book the Grand Plaza. Check-in March 15, check-out March 17.
Agent: Here's your booking summary:
  - Hotel: Grand Plaza, Colombo
  - Room: Deluxe
  - Check-in: March 15 | Check-out: March 17
  - Guests: 2
  - Total: $300.00
  Shall I confirm this booking?

You: Yes, please confirm.
Agent: Your booking is confirmed!
  Confirmation ID: BK-A3F29E1C
  Hotel: Grand Plaza, Colombo
  March 15 - March 17 | 2 guests | $300.00
```

You can also test via curl:

```bash
curl -X POST http://localhost:8080/booking/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "Find hotels in Kandy", "sessionId": "user-1"}'
```

## Step 7: Debug with tracing

If the agent selects the wrong tool or produces unexpected responses:

1. Press `Ctrl/Cmd` + `Shift` + `P` and run **`Ballerina: Enable Tracing`**.
2. Re-run the integration and interact with the agent.
3. Click **Show Logs** under any response to see the full trace: LLM calls, tool selections, and parameter values.

## What's Next

- [Adding Tools to an Agent](../../build-ai-applications/ai-agents/adding-tools-to-an-agent.md) -- Connect agents to external services and APIs
- [Adding Memory to an Agent](../../build-ai-applications/ai-agents/adding-memory-to-an-agent.md) -- Configure conversation, semantic, and persistent memory
- [Advanced AI Agent Configurations](../../build-ai-applications/ai-agents/agent-configuration-options.md) -- Multi-agent orchestration and advanced patterns
