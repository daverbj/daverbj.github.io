---
layout: post
title: "Building Real-Time Phone Call Transcription with Twilio, Next.js, and Vosk"
date: 2025-12-29 12:00:00 +0000
categories: twilio nextjs speech-recognition vosk real-time
---

Learn how to build a browser-based phone system with live speech-to-text transcription using Twilio Voice SDK, Next.js, and the open-source Vosk speech recognition engine.

## Table of Contents

1. [Introduction](#introduction)
2. [What We're Building](#what-were-building)
3. [Prerequisites](#prerequisites)
4. [Architecture Overview](#architecture-overview)
5. [Project Structure](#project-structure)
6. [Part 1: Setting Up Twilio](#part-1-setting-up-twilio)
7. [Part 2: Building the Next.js Client](#part-2-building-the-nextjs-client)
8. [Part 3: Building the Python Transcription Server](#part-3-building-the-python-transcription-server)
9. [Part 4: Connecting Everything Together](#part-4-connecting-everything-together)
10. [Part 5: Running the Application](#part-5-running-the-application)
11. [How It Works](#how-it-works)
12. [Troubleshooting](#troubleshooting)
13. [Next Steps](#next-steps)
14. [Conclusion](#conclusion)

---

## Introduction

Real-time transcription of phone calls has numerous applications—from call centers and customer support to accessibility features and meeting documentation. While cloud-based transcription services like Google Speech-to-Text or AWS Transcribe are popular choices, they can be expensive at scale and raise privacy concerns for sensitive conversations.

In this tutorial, we'll build a complete browser-based phone system that transcribes calls in real-time using **Vosk**, an open-source, offline speech recognition toolkit. This approach gives you:

- **Privacy**: Audio is processed locally, never leaving your server
- **Cost-effectiveness**: No per-minute transcription fees
- **Low latency**: Direct processing without cloud round-trips
- **Full control**: Customize the speech model for your use case

---

## Use Cases

Real-time call transcription with local processing opens up powerful possibilities across industries. Here are the most impactful use cases:

### Healthcare & HIPAA Compliance

**The Challenge**: Healthcare organizations must comply with HIPAA (Health Insurance Portability and Accountability Act), which mandates strict controls over Protected Health Information (PHI). Sending patient call audio to third-party cloud services creates compliance risks.

**The Solution**: Local transcription keeps all audio and transcripts on your own infrastructure:

- **Telemedicine Calls**: Transcribe doctor-patient consultations without PHI leaving your network
- **Patient Support Lines**: Document calls while maintaining HIPAA compliance
- **Medical Dictation**: Real-time transcription for clinical notes
- **Insurance Claim Calls**: Capture details without third-party exposure

```
+-----------------------------------------------------------+
|                   HIPAA-Compliant Setup                   |
|                                                           |
|   Patient --> Twilio --> Your HIPAA Server --> PHI DB     |
|                              |                            |
|                         Local Vosk                        |
|                       (No Cloud APIs)                     |
|                                                           |
|   [OK] Audio never leaves your infrastructure             |
|   [OK] Transcripts stored in your compliant database      |
|   [OK] Full audit trail under your control                |
+-----------------------------------------------------------+
```

**Compliance Benefits**:
- No Business Associate Agreement (BAA) needed for transcription services
- Complete audit trail of all processing
- Data residency control—keep everything in your region
- Encryption at rest and in transit under your control

### Financial Services & PCI-DSS

**Use Cases**:
- **Banking Support**: Transcribe customer service calls without exposing account details
- **Trading Desks**: Real-time compliance monitoring for recorded lines
- **Insurance Claims**: Document conversations with sensitive financial information
- **Debt Collection**: Maintain compliant records of all communications

### Legal & Regulatory

**Use Cases**:
- **Attorney-Client Calls**: Privilege-protected transcription
- **Depositions**: Real-time court reporter assistance
- **Compliance Hotlines**: Document whistleblower calls securely
- **Contract Negotiations**: Searchable records of verbal agreements

### Call Centers & Customer Support

**Use Cases**:
- **Real-Time Agent Assist**: Surface relevant knowledge base articles as customers speak
- **Quality Assurance**: Automated scoring and compliance checking
- **Sentiment Analysis**: Detect frustrated customers for supervisor escalation
- **Training**: Create transcripts for coaching and onboarding

**Example Workflow**:
```
Customer: "I've been waiting for three weeks and nobody has called me back!"
                                    |
                                    v
                      +-----------------------------+
                      |  Sentiment: NEGATIVE -0.8   |
                      |  Topic: Callback            |
                      |  Priority: HIGH             |
                      +-----------------------------+
                                    |
                                    v
              +---------------------------------------------+
              | [ALERT] Alert: Escalate to supervisor       |
              | [INFO] Suggested: Offer compensation        |
              +---------------------------------------------+
```

### Education & Accessibility

**Use Cases**:
- **Live Captioning**: Real-time subtitles for deaf/hard-of-hearing students
- **Lecture Recording**: Searchable transcripts of online classes
- **Language Learning**: Pronunciation feedback with transcription comparison
- **Parent-Teacher Calls**: Document IEP meetings and discussions

### Enterprise & Internal Communications

**Use Cases**:
- **Meeting Transcription**: Searchable records of conference calls
- **HR Interviews**: Documented hiring conversations
- **Sales Calls**: Capture customer requirements and objections
- **Executive Briefings**: Quick-reference summaries of verbal updates

### Government & Defense

**Use Cases**:
- **Classified Communications**: On-premise processing with no external dependencies
- **Emergency Services**: 911 call transcription for dispatch assistance
- **Public Hearings**: Accessible records of government proceedings
- **Veteran Services**: HIPAA-compliant support line documentation

### Multilingual Support

With Vosk and Whisper supporting 99+ languages:

- **Global Customer Support**: Transcribe calls in any language
- **Translation Pipelines**: Transcribe → Translate → Respond
- **Immigration Services**: Document interviews in native languages
- **International Business**: Cross-border call documentation

### Emerging Use Cases

| Use Case | Description |
|----------|-------------|
| **Voice Biometrics** | Speaker identification from transcription metadata |
| **Fraud Detection** | Real-time pattern matching on call transcripts |
| **AI Agents** | Feed transcriptions to LLMs for automated responses |
| **Predictive Analytics** | Analyze call patterns for business intelligence |
| **Voice Search** | Make call recordings searchable by content |

### Why Local Processing Matters

| Concern | Cloud Transcription | Local Transcription |
|---------|--------------------|--------------------|
| **Data Privacy** | Data sent to third party | Data stays on your servers |
| **Compliance** | Requires BAAs, audits | Full control, simpler compliance |
| **Cost** | Per-minute pricing | Fixed infrastructure cost |
| **Latency** | Network round-trip | Direct processing |
| **Availability** | Depends on provider | Works offline |
| **Customization** | Limited | Full model control |

---

By the end of this tutorial, you'll have a web application that can:

- Make outbound calls from your browser to any phone number  
- Receive inbound calls from external phones  
- Display real-time transcriptions with speaker identification  
- Select audio input/output devices  
- Show a professional dial pad interface  

## Prerequisites

Before we begin, make sure you have:

- **Node.js 18+** installed
- **Python 3.10+** installed
- **A Twilio account** ([Sign up free](https://www.twilio.com/try-twilio))
- **A Twilio phone number** (available in free trial)
- **A public URL** for webhooks (we'll use Cloudflare Tunnel, but ngrok works too)
- Basic familiarity with React, TypeScript, and Python

## Architecture Overview

Our application consists of three main components working together:

```
+-----------------------------------------------------------------------+
|                       YOUR DOMAIN (port 3000)                         |
|                                                                       |
|  +----------------------------------------------------------------+   |
|  |                     Python Flask Server                        |   |
|  |                                                                |   |
|  |   /media-stream -------> Vosk Speech Recognition               |   |
|  |        ^                        |                              |   |
|  |        |                        v                              |   |
|  |   Twilio Audio            Transcription                        |   |
|  |   (mulaw 8kHz)                  |                              |   |
|  |                                 v                              |   |
|  |   /transcription <-------- Send to Browser                     |   |
|  |        |                                                       |   |
|  |        v                                                       |   |
|  |   Browser WebSocket                                            |   |
|  |                                                                |   |
|  |   /* (all other routes) -------> Next.js (port 3001)           |   |
|  |                                                                |   |
|  +----------------------------------------------------------------+   |
|                                                                       |
+-----------------------------------------------------------------------+

+----------------+         +----------------+         +----------------+
|    Browser     | <-----> |    Twilio      | <-----> |  Phone/PSTN    |
|  (Voice SDK)   |         |    Cloud       |         |                |
+----------------+         +----------------+         +----------------+
```

**Data Flow:**

1. Browser initiates call using Twilio Voice SDK
2. Twilio connects the call and streams audio to our `/media-stream` WebSocket
3. Python server receives audio, converts it, and feeds it to Vosk
4. Vosk transcribes the speech and sends text to connected browsers via `/transcription` WebSocket
5. React UI displays the transcriptions in real-time

## Project Structure

Here's the complete project structure we'll create:

```
twiliorealtime/
+-- client/                          # Next.js application
|   +-- app/
|   |   +-- api/
|   |   |   +-- twilio/
|   |   |       +-- token/
|   |   |       |   +-- route.ts     # Generate Twilio access tokens
|   |   |       +-- voice/
|   |   |           +-- route.ts     # TwiML for call routing
|   |   +-- components/
|   |   |   +-- TwilioPhone.tsx      # Main phone UI component
|   |   +-- hooks/
|   |   |   +-- useTranscription.ts  # WebSocket hook for transcriptions
|   |   +-- globals.css
|   |   +-- layout.tsx
|   |   +-- page.tsx
|   +-- .env.local                   # Environment variables
|   +-- package.json
|   +-- tsconfig.json
|
+-- transcription-server/            # Python Vosk server
    +-- app.py                       # Main server with WebSocket handlers
    +-- requirements.txt             # Python dependencies
    +-- model/                       # Vosk speech model (downloaded)
    +-- venv/                        # Python virtual environment
```

---

## Part 1: Setting Up Twilio

### Step 1.1: Create a TwiML Application

A TwiML Application tells Twilio where to send webhook requests when calls are made.

1. Log into [Twilio Console](https://console.twilio.com)
2. Navigate to **Voice** → **Manage** → **TwiML Apps**
3. Click **Create new TwiML App**
4. Configure:
   - **Friendly Name**: `Browser Calling App`
   - **Voice Request URL**: `https://your-domain.com/api/twilio/voice`
   - **Voice Request Method**: `POST`
5. Click **Create**
6. Copy the **SID** (starts with `AP...`)

### Step 1.2: Create API Credentials

We need API keys for secure token generation:

1. Go to **Account** → **API keys & tokens**
2. Click **Create API Key**
3. Configure:
   - **Friendly Name**: `Browser Calling Key`
   - **Key Type**: `Standard`
4. Click **Create API Key**
5. **Important**: Copy both the **SID** and **Secret** immediately (the secret is only shown once!)

### Step 1.3: Get Your Twilio Phone Number

1. Go to **Phone Numbers** → **Manage** → **Active Numbers**
2. If you don't have a number, click **Buy a Number**
3. Configure your number:
   - **A Call Comes In**: Webhook → `https://your-domain.com/api/twilio/voice` (POST)
4. Copy your phone number (e.g., `+1XXXXXXXXXX`)

### Step 1.4: Gather Your Credentials

You should now have:

| Credential | Example | Where to Find |
|------------|---------|---------------|
| Account SID | `ACxxxx...` | Console Dashboard |
| API Key SID | `SKxxxx...` | API keys page |
| API Key Secret | `xxxxxx...` | Shown once when created |
| TwiML App SID | `APxxxx...` | TwiML Apps page |
| Phone Number | `+1XXXXXXXXXX` | Active Numbers page |

---

## Part 2: Building the Next.js Client

### Step 2.1: Initialize the Project

```bash
mkdir twiliorealtime && cd twiliorealtime
npx create-next-app@latest client --typescript --tailwind --app --src-dir=false
cd client
```

### Step 2.2: Install Dependencies

```bash
npm install @twilio/voice-sdk twilio
```

### Step 2.3: Configure Environment Variables

Create `client/.env.local`:

```env
# Twilio Credentials
TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_API_KEY_SID=SKxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_API_KEY_SECRET=your_api_key_secret_here
TWILIO_TWIML_APP_SID=APxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_PHONE_NUMBER=+1XXXXXXXXXX

# Your public domain (Cloudflare Tunnel, ngrok, etc.)
NEXT_PUBLIC_BASE_URL=https://your-domain.com

# WebSocket URL for transcriptions (same domain)
VOSK_WS_URL=wss://your-domain.com/media-stream
NEXT_PUBLIC_TRANSCRIPTION_WS_URL=wss://your-domain.com/transcription
```

### Step 2.4: Create the Token API Endpoint

The Twilio Voice SDK needs an access token to authenticate. Create `app/api/twilio/token/route.ts`:

```typescript
import { NextResponse } from "next/server";
import twilio from "twilio";

const accountSid = process.env.TWILIO_ACCOUNT_SID!;
const apiKeySid = process.env.TWILIO_API_KEY_SID!;
const apiKeySecret = process.env.TWILIO_API_KEY_SECRET!;
const twimlAppSid = process.env.TWILIO_TWIML_APP_SID!;

export async function GET() {
  try {
    // Create an access token
    const AccessToken = twilio.jwt.AccessToken;
    const VoiceGrant = AccessToken.VoiceGrant;

    const token = new AccessToken(accountSid, apiKeySid, apiKeySecret, {
      identity: "browser", // Unique identifier for this client
      ttl: 3600, // Token expires in 1 hour
    });

    // Grant voice capabilities
    const voiceGrant = new VoiceGrant({
      outgoingApplicationSid: twimlAppSid,
      incomingAllow: true, // Allow incoming calls
    });

    token.addGrant(voiceGrant);

    return NextResponse.json({
      token: token.toJwt(),
      identity: "browser",
    });
  } catch (error) {
    console.error("Error generating token:", error);
    return NextResponse.json(
      { error: "Failed to generate token" },
      { status: 500 }
    );
  }
}
```

### Step 2.5: Create the Voice Webhook Endpoint

This endpoint returns TwiML instructions for Twilio. Create `app/api/twilio/voice/route.ts`:

```typescript
import { NextRequest, NextResponse } from "next/server";
import twilio from "twilio";

const twilioPhoneNumber = process.env.TWILIO_PHONE_NUMBER;
const voskWsUrl = process.env.VOSK_WS_URL || "wss://your-domain.com/media-stream";

export async function POST(request: NextRequest) {
  try {
    const formData = await request.formData();
    const to = formData.get("To") as string;
    const from = formData.get("From") as string;
    const callSid = formData.get("CallSid") as string;

    console.log("Voice webhook:", { to, from, callSid });

    const VoiceResponse = twilio.twiml.VoiceResponse;
    const twiml = new VoiceResponse();

    // Start streaming audio to our Vosk transcription server
    const start = twiml.start();
    start.stream({
      url: voskWsUrl,
      track: "both_tracks", // Capture both caller and callee audio
    });

    const isFromClient = from?.startsWith("client:");
    const isToPhoneNumber = to?.startsWith("+");

    if (isFromClient && isToPhoneNumber) {
      // Outbound call from browser to phone number
      console.log("Outbound call to:", to);
      const dial = twiml.dial({
        callerId: twilioPhoneNumber,
        answerOnBridge: true,
        timeout: 30,
      });
      dial.number(to);
    } else if (!isFromClient && to === twilioPhoneNumber) {
      // Inbound call from phone to our Twilio number
      console.log("Inbound call from:", from);
      const dial = twiml.dial({
        callerId: from,
        answerOnBridge: true,
        timeout: 30,
      });
      dial.client("browser"); // Route to our browser client
    } else {
      twiml.say("Thanks for calling! Goodbye.");
    }

    console.log("TwiML:", twiml.toString());

    return new NextResponse(twiml.toString(), {
      headers: { "Content-Type": "text/xml" },
    });
  } catch (error) {
    console.error("Voice webhook error:", error);
    const twiml = new twilio.twiml.VoiceResponse();
    twiml.say("An error occurred. Please try again.");
    return new NextResponse(twiml.toString(), {
      headers: { "Content-Type": "text/xml" },
    });
  }
}
```

### Step 2.6: Create the Transcription Hook

This custom hook manages the WebSocket connection for receiving transcriptions. Create `app/hooks/useTranscription.ts`:

```typescript
"use client";

import { useState, useEffect, useCallback, useRef } from "react";

export interface TranscriptionEntry {
  id: string;
  text: string;
  timestamp: string;
  speaker?: "customer" | "agent";
  isFinal?: boolean;
}

function getDefaultWsUrl(): string {
  if (process.env.NEXT_PUBLIC_TRANSCRIPTION_WS_URL) {
    return process.env.NEXT_PUBLIC_TRANSCRIPTION_WS_URL;
  }
  if (typeof window !== "undefined") {
    const protocol = window.location.protocol === "https:" ? "wss:" : "ws:";
    return `${protocol}//${window.location.host}/transcription`;
  }
  return "ws://localhost:3000/transcription";
}

export function useTranscription() {
  const [transcriptions, setTranscriptions] = useState<TranscriptionEntry[]>([]);
  const [isConnected, setIsConnected] = useState(false);
  const [isTranscribing, setIsTranscribing] = useState(false);
  
  const wsRef = useRef<WebSocket | null>(null);
  const callSidRef = useRef<string | null>(null);
  const heartbeatRef = useRef<NodeJS.Timeout | null>(null);

  const startHeartbeat = useCallback((ws: WebSocket) => {
    heartbeatRef.current = setInterval(() => {
      if (ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify({ type: "ping" }));
      }
    }, 15000);
  }, []);

  const stopHeartbeat = useCallback(() => {
    if (heartbeatRef.current) {
      clearInterval(heartbeatRef.current);
      heartbeatRef.current = null;
    }
  }, []);

  const connect = useCallback((callSid: string) => {
    callSidRef.current = callSid;

    if (wsRef.current?.readyState === WebSocket.OPEN) {
      wsRef.current.send(JSON.stringify({ type: "register", callSid }));
      return;
    }

    const ws = new WebSocket(getDefaultWsUrl());

    ws.onopen = () => {
      setIsConnected(true);
      startHeartbeat(ws);
      if (callSidRef.current) {
        ws.send(JSON.stringify({ type: "register", callSid: callSidRef.current }));
      }
    };

    ws.onmessage = (event) => {
      const message = JSON.parse(event.data);

      switch (message.type) {
        case "pong":
          break;
        case "transcription_started":
          setIsTranscribing(true);
          break;
        case "transcription":
          const entry: TranscriptionEntry = {
            id: `${message.callSid}-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
            text: message.text,
            timestamp: message.timestamp,
            speaker: message.speaker,
            isFinal: message.isFinal,
          };
          setTranscriptions((prev) => [...prev, entry]);
          break;
        case "transcription_ended":
          setIsTranscribing(false);
          break;
      }
    };

    ws.onclose = () => {
      setIsConnected(false);
      setIsTranscribing(false);
      stopHeartbeat();
    };

    wsRef.current = ws;
  }, [startHeartbeat, stopHeartbeat]);

  const disconnect = useCallback(() => {
    stopHeartbeat();
    setTimeout(() => {
      wsRef.current?.close();
      wsRef.current = null;
      setIsConnected(false);
      setIsTranscribing(false);
    }, 2000); // Wait for final transcriptions
  }, [stopHeartbeat]);

  const clearTranscriptions = useCallback(() => {
    setTranscriptions([]);
  }, []);

  useEffect(() => {
    return () => {
      stopHeartbeat();
      wsRef.current?.close();
    };
  }, [stopHeartbeat]);

  return {
    transcriptions,
    isConnected,
    isTranscribing,
    connect,
    disconnect,
    clearTranscriptions,
  };
}
```

### Step 2.7: Create the Phone UI Component

This is the main component with dial pad and transcription display. Create `app/components/TwilioPhone.tsx`:

```typescript
"use client";

import { useState, useEffect, useRef, useCallback } from "react";
import { Device, Call } from "@twilio/voice-sdk";
import { useTranscription } from "../hooks/useTranscription";

type CallState = "idle" | "connecting" | "ringing" | "connected" | "disconnected";

export default function TwilioPhone() {
  const [device, setDevice] = useState<Device | null>(null);
  const [call, setCall] = useState<Call | null>(null);
  const [callState, setCallState] = useState<CallState>("idle");
  const [phoneNumber, setPhoneNumber] = useState("");
  const [audioDevices, setAudioDevices] = useState<MediaDeviceInfo[]>([]);
  const [selectedDevice, setSelectedDevice] = useState<string>("");
  const [incomingCall, setIncomingCall] = useState<Call | null>(null);

  const {
    transcriptions,
    isConnected: wsConnected,
    isTranscribing,
    connect: connectWs,
    disconnect: disconnectWs,
    clearTranscriptions,
  } = useTranscription();

  const transcriptionEndRef = useRef<HTMLDivElement>(null);

  // Auto-scroll transcriptions
  useEffect(() => {
    transcriptionEndRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [transcriptions]);

  // Initialize Twilio Device
  useEffect(() => {
    async function initDevice() {
      try {
        const response = await fetch("/api/twilio/token");
        const { token } = await response.json();

        const newDevice = new Device(token, {
          codecPreferences: [Call.Codec.PCMU, Call.Codec.Opus],
          enableImprovedSignalingErrorPrecision: true,
        });

        newDevice.on("registered", () => console.log("Device registered"));
        newDevice.on("error", (error) => console.error("Device error:", error));
        
        newDevice.on("incoming", (incomingCall) => {
          console.log("Incoming call from:", incomingCall.parameters.From);
          setIncomingCall(incomingCall);
        });

        await newDevice.register();
        setDevice(newDevice);

        // Get audio devices
        await newDevice.audio?.setAudioConstraints({ echoCancellation: true });
        const devices = await navigator.mediaDevices.enumerateDevices();
        const audioInputs = devices.filter((d) => d.kind === "audioinput");
        setAudioDevices(audioInputs);
        if (audioInputs.length > 0) {
          setSelectedDevice(audioInputs[0].deviceId);
        }
      } catch (error) {
        console.error("Failed to initialize device:", error);
      }
    }

    initDevice();
  }, []);

  const makeCall = useCallback(async () => {
    if (!device || !phoneNumber) return;

    try {
      setCallState("connecting");
      clearTranscriptions();

      const newCall = await device.connect({
        params: { To: phoneNumber },
      });

      newCall.on("accept", () => {
        setCallState("connected");
        const callSid = newCall.parameters.CallSid;
        if (callSid) connectWs(callSid);
      });

      newCall.on("disconnect", () => {
        setCallState("disconnected");
        disconnectWs();
        setCall(null);
        setTimeout(() => setCallState("idle"), 2000);
      });

      newCall.on("error", (error) => {
        console.error("Call error:", error);
        setCallState("idle");
      });

      setCall(newCall);
    } catch (error) {
      console.error("Failed to make call:", error);
      setCallState("idle");
    }
  }, [device, phoneNumber, clearTranscriptions, connectWs, disconnectWs]);

  const acceptCall = useCallback(() => {
    if (!incomingCall) return;

    incomingCall.accept();
    setCall(incomingCall);
    setCallState("connected");
    setIncomingCall(null);
    clearTranscriptions();

    const callSid = incomingCall.parameters.CallSid;
    if (callSid) connectWs(callSid);

    incomingCall.on("disconnect", () => {
      setCallState("disconnected");
      disconnectWs();
      setCall(null);
      setTimeout(() => setCallState("idle"), 2000);
    });
  }, [incomingCall, clearTranscriptions, connectWs, disconnectWs]);

  const hangUp = useCallback(() => {
    call?.disconnect();
    incomingCall?.reject();
    setIncomingCall(null);
  }, [call, incomingCall]);

  const dialPadPress = (digit: string) => {
    setPhoneNumber((prev) => prev + digit);
    call?.sendDigits(digit); // Send DTMF if on call
  };

  return (
    <div className="max-w-md mx-auto p-6 bg-white rounded-xl shadow-lg">
      <h1 className="text-2xl font-bold text-center mb-4">Twilio Phone</h1>

      {/* Phone Number Display */}
      <div className="mb-4">
        <input
          type="tel"
          value={phoneNumber}
          onChange={(e) => setPhoneNumber(e.target.value)}
          placeholder="+1XXXXXXXXXX"
          className="w-full p-3 text-xl text-center border rounded-lg"
          disabled={callState !== "idle"}
        />
      </div>

      {/* Dial Pad */}
      <div className="grid grid-cols-3 gap-2 mb-4">
        {["1", "2", "3", "4", "5", "6", "7", "8", "9", "*", "0", "#"].map((digit) => (
          <button
            key={digit}
            onClick={() => dialPadPress(digit)}
            className="p-4 text-xl font-semibold bg-gray-100 rounded-lg hover:bg-gray-200"
          >
            {digit}
          </button>
        ))}
      </div>

      {/* Call Controls */}
      <div className="flex gap-2 mb-4">
        {callState === "idle" && (
          <button
            onClick={makeCall}
            disabled={!phoneNumber}
            className="flex-1 p-3 bg-green-500 text-white rounded-lg hover:bg-green-600 disabled:opacity-50"
          >
            Call
          </button>
        )}
        {(callState === "connecting" || callState === "connected") && (
          <button
            onClick={hangUp}
            className="flex-1 p-3 bg-red-500 text-white rounded-lg hover:bg-red-600"
          >
            📵 Hang Up
          </button>
        )}
      </div>

      {/* Incoming Call Alert */}
      {incomingCall && (
        <div className="mb-4 p-4 bg-yellow-100 rounded-lg">
          <p className="font-semibold">Incoming call from: {incomingCall.parameters.From}</p>
          <div className="flex gap-2 mt-2">
            <button
              onClick={acceptCall}
              className="flex-1 p-2 bg-green-500 text-white rounded"
            >
              Accept
            </button>
            <button
              onClick={hangUp}
              className="flex-1 p-2 bg-red-500 text-white rounded"
            >
              Reject
            </button>
          </div>
        </div>
      )}

      {/* Call Status */}
      <div className="text-center text-sm text-gray-500 mb-4">
        Status: {callState} | WS: {wsConnected ? "🟢" : "🔴"} | Transcribing: {isTranscribing ? "🟢" : "⚪"}
      </div>

      {/* Transcription Panel */}
      <div className="border rounded-lg p-3 h-64 overflow-y-auto bg-gray-50">
        <h3 className="font-semibold mb-2">Live Transcription</h3>
        {transcriptions.length === 0 ? (
          <p className="text-gray-400 text-sm">Transcriptions will appear here...</p>
        ) : (
          transcriptions.map((t) => (
            <div key={t.id} className="mb-2">
              <span className={`font-semibold ${t.speaker === "agent" ? "text-blue-600" : "text-green-600"}`}>
                {t.speaker === "agent" ? "You" : "Customer"}:
              </span>
              <span className="ml-2">{t.text}</span>
            </div>
          ))
        )}
        <div ref={transcriptionEndRef} />
      </div>

      {/* Audio Device Selector */}
      <div className="mt-4">
        <label className="block text-sm text-gray-600 mb-1">Microphone:</label>
        <select
          value={selectedDevice}
          onChange={(e) => setSelectedDevice(e.target.value)}
          className="w-full p-2 border rounded"
        >
          {audioDevices.map((device) => (
            <option key={device.deviceId} value={device.deviceId}>
              {device.label || `Microphone ${device.deviceId.slice(0, 8)}`}
            </option>
          ))}
        </select>
      </div>
    </div>
  );
}
```

### Step 2.8: Update the Main Page

Replace `app/page.tsx`:

```typescript
import TwilioPhone from "./components/TwilioPhone";

export default function Home() {
  return (
    <main className="min-h-screen bg-gray-100 py-10">
      <TwilioPhone />
    </main>
  );
}
```

---

## Part 3: Building the Python Transcription Server

### Step 3.1: Create the Project Directory

```bash
cd ..  # Go back to twiliorealtime root
mkdir transcription-server
cd transcription-server
```

### Step 3.2: Set Up Python Virtual Environment

```bash
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

### Step 3.3: Create Requirements File

Create `requirements.txt`:

```
flask==3.0.0
flask-sock==0.7.0
simple-websocket==1.0.0
vosk==0.3.44
audioop-lts
requests
```

Install dependencies:

```bash
pip install -r requirements.txt
```

### Step 3.4: Download Vosk Model

Vosk requires a language model for speech recognition. Download the small English model:

```bash
curl -LO https://alphacephei.com/vosk/models/vosk-model-small-en-us-0.15.zip
unzip vosk-model-small-en-us-0.15.zip
mv vosk-model-small-en-us-0.15 model
rm vosk-model-small-en-us-0.15.zip
```

> **Tip**: For better accuracy in production, use a larger model like `vosk-model-en-us-0.22` (1.8GB).

### Step 3.5: Create the Main Server

Create `app.py`:

```python
import audioop
import base64
import json
import os
import threading
import requests
from flask import Flask, request, Response
from flask_sock import Sock, ConnectionClosed
import vosk

app = Flask(__name__)
sock = Sock(app)

# Next.js server URL (runs on port 3001)
NEXTJS_URL = os.environ.get('NEXTJS_URL', 'http://localhost:3001')

# Load Vosk model
SCRIPT_DIR = os.path.dirname(os.path.abspath(__file__))
MODEL_PATH = os.environ.get('VOSK_MODEL_PATH', os.path.join(SCRIPT_DIR, 'model'))

print(f"Loading Vosk model from: {MODEL_PATH}")
try:
    model = vosk.Model(MODEL_PATH)
    print("[OK] Vosk model loaded successfully!")
except Exception as e:
    print(f"[ERROR] Error loading Vosk model: {e}")
    model = None

# Store browser WebSocket connections by callSid
browser_connections = {}
pending_transcriptions = {}
connections_lock = threading.Lock()


def send_to_browser(call_sid, data):
    """Send transcription data to the connected browser."""
    with connections_lock:
        ws = browser_connections.get(call_sid)
        if ws:
            try:
                ws.send(json.dumps({
                    'type': 'transcription',
                    'callSid': call_sid,
                    **data
                }))
                print(f"[LOG] [{data.get('speaker', '?')}]: {data.get('text', '')}")
            except Exception as e:
                print(f"Error sending to browser: {e}")
        else:
            # Queue for later
            if call_sid not in pending_transcriptions:
                pending_transcriptions[call_sid] = []
            pending_transcriptions[call_sid].append(data)


@app.route('/health')
def health():
    """Health check endpoint."""
    return {'status': 'ok', 'model_loaded': model is not None}


@sock.route('/transcription')
def transcription_ws(ws):
    """WebSocket endpoint for browser to receive transcriptions."""
    print("[WS] Browser connected to transcription service")
    registered_call_sid = None
    
    try:
        while True:
            message = ws.receive()
            if message is None:
                break
                
            data = json.loads(message)
            
            if data.get('type') == 'ping':
                ws.send(json.dumps({'type': 'pong'}))
                
            elif data.get('type') == 'register':
                call_sid = data.get('callSid')
                if call_sid:
                    registered_call_sid = call_sid
                    with connections_lock:
                        browser_connections[call_sid] = ws
                    print(f"📱 Browser registered for call: {call_sid}")
                    
                    # Send pending transcriptions
                    pending = pending_transcriptions.pop(call_sid, [])
                    for item in pending:
                        ws.send(json.dumps({
                            'type': 'transcription',
                            'callSid': call_sid,
                            **item
                        }))
                    
                    ws.send(json.dumps({
                        'type': 'transcription_started',
                        'callSid': call_sid
                    }))
                    
    except ConnectionClosed:
        pass
    finally:
        print("[WS] Browser disconnected")
        if registered_call_sid:
            with connections_lock:
                if browser_connections.get(registered_call_sid) == ws:
                    del browser_connections[registered_call_sid]


@sock.route('/media-stream')
def media_stream(ws):
    """Receive and transcribe Twilio media stream audio."""
    if not model:
        print("[ERROR] Vosk model not loaded")
        return
        
    print("Twilio Media Stream connected")
    
    # Create recognizers for both tracks
    inbound_rec = vosk.KaldiRecognizer(model, 16000)
    outbound_rec = vosk.KaldiRecognizer(model, 16000)
    
    call_sid = None
    last_inbound = ""
    last_outbound = ""
    
    try:
        while True:
            message = ws.receive()
            if message is None:
                break
                
            packet = json.loads(message)
            event = packet.get('event')
            
            if event == 'start':
                start_data = packet.get('start', {})
                call_sid = start_data.get('callSid')
                print(f"🎙️ Stream started for call: {call_sid}")
                
            elif event == 'media':
                media = packet.get('media', {})
                track = media.get('track', 'inbound')
                payload = media.get('payload', '')
                
                if not payload:
                    continue
                
                # Decode and convert audio: base64 -> mulaw -> PCM -> 16kHz
                try:
                    audio = base64.b64decode(payload)
                    audio = audioop.ulaw2lin(audio, 2)
                    audio = audioop.ratecv(audio, 2, 1, 8000, 16000, None)[0]
                except Exception as e:
                    continue
                
                # Choose recognizer based on track
                if track == 'outbound':
                    rec = outbound_rec
                    speaker = 'agent'
                    last_ref = last_outbound
                else:
                    rec = inbound_rec
                    speaker = 'customer'
                    last_ref = last_inbound
                
                # Process audio through Vosk
                if rec.AcceptWaveform(audio):
                    result = json.loads(rec.Result())
                    text = result.get('text', '').strip()
                    
                    if text and text != last_ref and call_sid:
                        if track == 'outbound':
                            last_outbound = text
                        else:
                            last_inbound = text
                            
                        send_to_browser(call_sid, {
                            'text': text,
                            'speaker': speaker,
                            'isFinal': True,
                            'timestamp': media.get('timestamp', '')
                        })
                        
            elif event == 'stop':
                print(f"Stream stopped for call: {call_sid}")
                
                # Process remaining audio
                for rec, speaker in [(inbound_rec, 'customer'), (outbound_rec, 'agent')]:
                    final = json.loads(rec.FinalResult())
                    text = final.get('text', '').strip()
                    if text and call_sid:
                        send_to_browser(call_sid, {
                            'text': text,
                            'speaker': speaker,
                            'isFinal': True,
                            'timestamp': ''
                        })
                
                # Notify browser
                with connections_lock:
                    browser_ws = browser_connections.get(call_sid)
                    if browser_ws:
                        try:
                            browser_ws.send(json.dumps({
                                'type': 'transcription_ended',
                                'callSid': call_sid
                            }))
                        except:
                            pass
                break
                
    except ConnectionClosed:
        print("Twilio Media Stream closed")
    except Exception as e:
        print(f"[ERROR] Media stream error: {e}")


# Reverse proxy to Next.js
@app.route('/', defaults={'path': ''}, methods=['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS'])
@app.route('/<path:path>', methods=['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS'])
def proxy(path):
    """Proxy all non-WebSocket requests to Next.js."""
    if path in ['transcription', 'media-stream', 'health']:
        return {'error': 'Use WebSocket'}, 400
    
    target_url = f"{NEXTJS_URL}/{path}"
    if request.query_string:
        target_url += f"?{request.query_string.decode()}"
    
    try:
        headers = {k: v for k, v in request.headers if k.lower() not in [
            'host', 'connection', 'transfer-encoding', 'upgrade'
        ]}
        
        resp = requests.request(
            method=request.method,
            url=target_url,
            headers=headers,
            data=request.get_data(),
            cookies=request.cookies,
            allow_redirects=False,
            stream=True,
            timeout=60
        )
        
        excluded = ['content-encoding', 'content-length', 'transfer-encoding', 'connection']
        response_headers = [(n, v) for n, v in resp.raw.headers.items() if n.lower() not in excluded]
        
        return Response(resp.content, status=resp.status_code, headers=response_headers)
        
    except requests.exceptions.ConnectionError:
        return {'error': 'Next.js not running', 'hint': 'Run: cd client && PORT=3001 npm run dev'}, 502


if __name__ == '__main__':
    port = int(os.environ.get('PORT', 3000))
    print(f"""
+==============================================================+
|          Vosk Transcription Server                           |
+==============================================================+
|  WebSocket Endpoints:                                        |
|    • /media-stream  - Twilio audio stream                    |
|    • /transcription - Browser transcription feed             |
|                                                              |
|  All other requests proxy to Next.js at {NEXTJS_URL:<17} |
|                                                              |
|  Listening on port {port:<41}|
+==============================================================+
    """)
    app.run(host='0.0.0.0', port=port, debug=False)
```

---

## Part 4: Connecting Everything Together

### Step 4.1: Set Up Your Public URL

Twilio needs to reach your server from the internet. You have several options:

**Option A: Cloudflare Tunnel (Recommended)**

```bash
# Install cloudflared
brew install cloudflared  # macOS

# Create a tunnel
cloudflared tunnel login
cloudflared tunnel create my-tunnel
cloudflared tunnel route dns my-tunnel your-subdomain.your-domain.com

# Run the tunnel
cloudflared tunnel run --url http://localhost:3000 my-tunnel
```

**Option B: ngrok**

```bash
ngrok http 3000
```

### Step 4.2: Update Twilio Webhooks

1. Go to Twilio Console → Voice → TwiML Apps → Your App
2. Update the Voice URL to: `https://your-domain.com/api/twilio/voice`
3. Go to Phone Numbers → Your Number
4. Update "A Call Comes In" to: `https://your-domain.com/api/twilio/voice`

### Step 4.3: Update Environment Variables

Update `client/.env.local` with your actual domain:

```env
NEXT_PUBLIC_BASE_URL=https://your-actual-domain.com
VOSK_WS_URL=wss://your-actual-domain.com/media-stream
NEXT_PUBLIC_TRANSCRIPTION_WS_URL=wss://your-actual-domain.com/transcription
```

---

## Part 5: Running the Application

### Step 5.1: Start Next.js (Terminal 1)

```bash
cd client
PORT=3001 npm run dev
```

You should see:
```
^ Next.js 14.x.x
- Local:        http://localhost:3001
- Ready in 2.3s
```

### Step 5.2: Start Python Server (Terminal 2)

```bash
cd transcription-server
source venv/bin/activate
python app.py
```

You should see:
```
Loading Vosk model from: /path/to/model
- Vosk model loaded successfully!
+==============================================================+
|          Vosk Transcription Server                           |
+==============================================================+
|  Listening on port 3000                                      |
+==============================================================+
```

### Step 5.3: Start Your Tunnel (Terminal 3)

```bash
cloudflared tunnel run my-tunnel
# or
ngrok http 3000
```

### Step 5.4: Test the Application

1. Open `https://your-domain.com` in your browser
2. Enter a phone number (e.g., your mobile)
3. Click "Call"
4. Answer on your phone
5. Start speaking—you should see transcriptions appear in real-time!

---

## How It Works

Let's trace through what happens when you make a call:

### 1. Browser Initiates Call

```javascript
const call = await device.connect({ params: { To: "+1XXXXXXXXXX" } });
```

The Twilio Voice SDK sends a request to Twilio's servers.

### 2. Twilio Requests TwiML Instructions

Twilio makes a POST request to your webhook:

```
POST https://your-domain.com/api/twilio/voice
Body: To=+1XXXXXXXXXX&From=client:browser&CallSid=CA123...
```

### 3. Your Server Returns TwiML

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
  <Start>
    <Stream url="wss://your-domain.com/media-stream" track="both_tracks"/>
  </Start>
  <Dial callerId="+1XXXXXXXXXX" answerOnBridge="true">
    <Number>+1XXXXXXXXXX</Number>
  </Dial>
</Response>
```

### 4. Twilio Streams Audio to Your Server

Twilio opens a WebSocket to `/media-stream` and sends JSON messages:

```json
{"event": "start", "start": {"callSid": "CA123..."}}
{"event": "media", "media": {"track": "inbound", "payload": "base64audio..."}}
{"event": "media", "media": {"track": "outbound", "payload": "base64audio..."}}
```

### 5. Vosk Transcribes the Audio

```python
# Decode: base64 → mulaw → PCM → resample to 16kHz
audio = base64.b64decode(payload)
audio = audioop.ulaw2lin(audio, 2)
audio = audioop.ratecv(audio, 2, 1, 8000, 16000, None)[0]

# Transcribe
if recognizer.AcceptWaveform(audio):
    result = recognizer.Result()  # {"text": "hello world"}
```

### 6. Transcription Sent to Browser

The Python server sends the transcription via the `/transcription` WebSocket:

```json
{"type": "transcription", "callSid": "CA123...", "text": "hello world", "speaker": "customer"}
```

### 7. React Updates the UI

```javascript
// In useTranscription hook
ws.onmessage = (event) => {
  const message = JSON.parse(event.data);
  if (message.type === "transcription") {
    setTranscriptions(prev => [...prev, message]);
  }
};
```

---

## Troubleshooting

### "Vosk model not found"

Make sure you downloaded and extracted the model:

```bash
ls transcription-server/model/
# Should show: am/ conf/ graph/ ivector/ README
```

### "WebSocket connection failed"

1. Check that your tunnel is running
2. Verify the WebSocket URLs in `.env.local` match your domain
3. Check browser console for CORS errors

### "No audio/transcription"

1. Verify Twilio is reaching your webhook (check Twilio Console → Monitor → Calls)
2. Check Python server logs for incoming audio
3. Ensure the Vosk model is loaded (look for "[OK] Vosk model loaded")

### "Call connects but no one can hear"

1. Check that `answerOnBridge: true` is set in TwiML
2. Verify your Twilio number has voice capabilities
3. Test with a simple TwiML that just says "Hello"

### Poor transcription accuracy

1. Use a larger Vosk model (e.g., `vosk-model-en-us-0.22`)
2. Ensure good audio quality (minimize background noise)
3. Consider using a different speech recognition service for production

---

## Next Steps

Now that you have a working prototype, consider these enhancements:

### 1. Add Authentication
Protect your API endpoints with proper authentication (JWT, session cookies, etc.)

### 2. Store Transcriptions
Save transcriptions to a database for later review:

```python
# In send_to_browser()
db.save_transcription(call_sid, text, speaker, timestamp)
```

### 3. Use a Larger Model
For production, download a more accurate model:

```bash
curl -LO https://alphacephei.com/vosk/models/vosk-model-en-us-0.22.zip
```

### 4. Add Call Recording
Twilio can record calls alongside transcription:

```xml
<Dial record="record-from-answer-dual" recordingStatusCallback="/api/recording">
```

### 5. Deploy to Production
- Use Gunicorn or uWSGI for the Python server
- Deploy Next.js to Vercel
- Use a proper domain with SSL

### 6. Add Real-Time Translation
Pipe transcriptions through a translation API for multilingual support.

---

## Alternative: Using OpenAI Whisper Locally

While Vosk is lightweight and fast, OpenAI's **Whisper** model often provides better accuracy, especially for diverse accents and noisy environments. Here's how to use Whisper locally instead of (or alongside) Vosk.

### Why Whisper?

| Feature | Vosk | Whisper |
|---------|------|---------|
| Speed | ⚡ Very fast (real-time) | 🐢 Slower (batch processing) |
| Accuracy | Good | Excellent |
| Languages | 20+ | 99+ |
| Model Size | 40MB - 1.8GB | 39MB - 2.9GB |
| Best For | Real-time streaming | High accuracy |

### Step 1: Install Whisper

Add to your `requirements.txt`:

```
openai-whisper
torch
```

Or install directly:

```bash
pip install openai-whisper torch
```

> **Note**: Whisper requires PyTorch. For GPU acceleration, install the CUDA version of PyTorch.

### Step 2: Create a Whisper Transcription Server

Create `app_whisper.py`:

```python
import audioop
import base64
import json
import os
import threading
import queue
import tempfile
import wave
import requests
from flask import Flask, request, Response
from flask_sock import Sock, ConnectionClosed
import whisper

app = Flask(__name__)
sock = Sock(app)

NEXTJS_URL = os.environ.get('NEXTJS_URL', 'http://localhost:3001')

# Load Whisper model
# Options: tiny, base, small, medium, large, large-v2, large-v3
MODEL_SIZE = os.environ.get('WHISPER_MODEL', 'base')
print(f"Loading Whisper model: {MODEL_SIZE}")
model = whisper.load_model(MODEL_SIZE)
print(f"[OK] Whisper {MODEL_SIZE} model loaded!")

browser_connections = {}
pending_transcriptions = {}
connections_lock = threading.Lock()


def send_to_browser(call_sid, data):
    """Send transcription data to the connected browser."""
    with connections_lock:
        ws = browser_connections.get(call_sid)
        if ws:
            try:
                ws.send(json.dumps({
                    'type': 'transcription',
                    'callSid': call_sid,
                    **data
                }))
                print(f"[LOG] [{data.get('speaker', '?')}]: {data.get('text', '')}")
            except Exception as e:
                print(f"Error sending to browser: {e}")
        else:
            if call_sid not in pending_transcriptions:
                pending_transcriptions[call_sid] = []
            pending_transcriptions[call_sid].append(data)


class WhisperTranscriber:
    """Buffer audio and transcribe in chunks using Whisper."""
    
    def __init__(self, model, chunk_duration=3.0, sample_rate=16000):
        self.model = model
        self.chunk_duration = chunk_duration  # Seconds of audio before transcribing
        self.sample_rate = sample_rate
        self.buffer = bytearray()
        self.samples_per_chunk = int(chunk_duration * sample_rate)
        self.bytes_per_chunk = self.samples_per_chunk * 2  # 16-bit audio
        
    def add_audio(self, audio_bytes):
        """Add audio to buffer and return transcription if chunk is ready."""
        self.buffer.extend(audio_bytes)
        
        if len(self.buffer) >= self.bytes_per_chunk:
            # Extract chunk and transcribe
            chunk = bytes(self.buffer[:self.bytes_per_chunk])
            self.buffer = self.buffer[self.bytes_per_chunk:]
            
            return self._transcribe_chunk(chunk)
        return None
    
    def flush(self):
        """Transcribe any remaining audio in buffer."""
        if len(self.buffer) > self.sample_rate:  # At least 0.5 seconds
            result = self._transcribe_chunk(bytes(self.buffer))
            self.buffer.clear()
            return result
        self.buffer.clear()
        return None
    
    def _transcribe_chunk(self, audio_bytes):
        """Transcribe a chunk of audio using Whisper."""
        try:
            # Save to temporary WAV file
            with tempfile.NamedTemporaryFile(suffix='.wav', delete=False) as f:
                with wave.open(f.name, 'wb') as wav:
                    wav.setnchannels(1)
                    wav.setsampwidth(2)
                    wav.setframerate(self.sample_rate)
                    wav.writeframes(audio_bytes)
                
                # Transcribe
                result = self.model.transcribe(
                    f.name,
                    language='en',
                    fp16=False,  # Set True if using GPU
                    task='transcribe'
                )
                
                # Clean up
                os.unlink(f.name)
                
                text = result.get('text', '').strip()
                return text if text else None
                
        except Exception as e:
            print(f"Whisper transcription error: {e}")
            return None


@sock.route('/transcription')
def transcription_ws(ws):
    """WebSocket endpoint for browser to receive transcriptions."""
    print("[WS] Browser connected to transcription service")
    registered_call_sid = None
    
    try:
        while True:
            message = ws.receive()
            if message is None:
                break
                
            data = json.loads(message)
            
            if data.get('type') == 'ping':
                ws.send(json.dumps({'type': 'pong'}))
                
            elif data.get('type') == 'register':
                call_sid = data.get('callSid')
                if call_sid:
                    registered_call_sid = call_sid
                    with connections_lock:
                        browser_connections[call_sid] = ws
                    print(f"📱 Browser registered for call: {call_sid}")
                    
                    pending = pending_transcriptions.pop(call_sid, [])
                    for item in pending:
                        ws.send(json.dumps({
                            'type': 'transcription',
                            'callSid': call_sid,
                            **item
                        }))
                    
                    ws.send(json.dumps({
                        'type': 'transcription_started',
                        'callSid': call_sid
                    }))
                    
    except ConnectionClosed:
        pass
    finally:
        print("[WS] Browser disconnected")
        if registered_call_sid:
            with connections_lock:
                if browser_connections.get(registered_call_sid) == ws:
                    del browser_connections[registered_call_sid]


@sock.route('/media-stream')
def media_stream(ws):
    """Receive and transcribe Twilio media stream audio using Whisper."""
    print("Twilio Media Stream connected (Whisper)")
    
    # Create transcribers for both tracks
    inbound_transcriber = WhisperTranscriber(model, chunk_duration=3.0)
    outbound_transcriber = WhisperTranscriber(model, chunk_duration=3.0)
    
    call_sid = None
    
    try:
        while True:
            message = ws.receive()
            if message is None:
                break
                
            packet = json.loads(message)
            event = packet.get('event')
            
            if event == 'start':
                call_sid = packet.get('start', {}).get('callSid')
                print(f"🎙️ Stream started for call: {call_sid}")
                
            elif event == 'media':
                media = packet.get('media', {})
                track = media.get('track', 'inbound')
                payload = media.get('payload', '')
                
                if not payload:
                    continue
                
                # Decode and convert audio
                try:
                    audio = base64.b64decode(payload)
                    audio = audioop.ulaw2lin(audio, 2)
                    audio = audioop.ratecv(audio, 2, 1, 8000, 16000, None)[0]
                except Exception as e:
                    continue
                
                # Choose transcriber based on track
                if track == 'outbound':
                    transcriber = outbound_transcriber
                    speaker = 'agent'
                else:
                    transcriber = inbound_transcriber
                    speaker = 'customer'
                
                # Add audio and check for transcription
                text = transcriber.add_audio(audio)
                if text and call_sid:
                    send_to_browser(call_sid, {
                        'text': text,
                        'speaker': speaker,
                        'isFinal': True,
                        'timestamp': media.get('timestamp', '')
                    })
                        
            elif event == 'stop':
                print(f"Stream stopped for call: {call_sid}")
                
                # Flush remaining audio
                for transcriber, speaker in [
                    (inbound_transcriber, 'customer'),
                    (outbound_transcriber, 'agent')
                ]:
                    text = transcriber.flush()
                    if text and call_sid:
                        send_to_browser(call_sid, {
                            'text': text,
                            'speaker': speaker,
                            'isFinal': True,
                            'timestamp': ''
                        })
                
                with connections_lock:
                    browser_ws = browser_connections.get(call_sid)
                    if browser_ws:
                        try:
                            browser_ws.send(json.dumps({
                                'type': 'transcription_ended',
                                'callSid': call_sid
                            }))
                        except:
                            pass
                break
                
    except ConnectionClosed:
        print("Twilio Media Stream closed")
    except Exception as e:
        print(f"[ERROR] Media stream error: {e}")


# Reverse proxy to Next.js (same as before)
@app.route('/', defaults={'path': ''}, methods=['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS'])
@app.route('/<path:path>', methods=['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS'])
def proxy(path):
    if path in ['transcription', 'media-stream', 'health']:
        return {'error': 'Use WebSocket'}, 400
    
    target_url = f"{NEXTJS_URL}/{path}"
    if request.query_string:
        target_url += f"?{request.query_string.decode()}"
    
    try:
        headers = {k: v for k, v in request.headers if k.lower() not in [
            'host', 'connection', 'transfer-encoding', 'upgrade'
        ]}
        resp = requests.request(
            method=request.method,
            url=target_url,
            headers=headers,
            data=request.get_data(),
            cookies=request.cookies,
            allow_redirects=False,
            stream=True,
            timeout=60
        )
        excluded = ['content-encoding', 'content-length', 'transfer-encoding', 'connection']
        response_headers = [(n, v) for n, v in resp.raw.headers.items() if n.lower() not in excluded]
        return Response(resp.content, status=resp.status_code, headers=response_headers)
    except requests.exceptions.ConnectionError:
        return {'error': 'Next.js not running'}, 502


if __name__ == '__main__':
    port = int(os.environ.get('PORT', 3000))
    print(f"\nWhisper Transcription Server running on port {port}\n")
    app.run(host='0.0.0.0', port=port, debug=False)
```

### Step 3: Run with Whisper

```bash
# Use a specific model size
WHISPER_MODEL=small python app_whisper.py

# Or for better accuracy (requires more RAM/GPU)
WHISPER_MODEL=medium python app_whisper.py
```

### Whisper Model Comparison

| Model | Size | RAM Required | Relative Speed | Accuracy |
|-------|------|--------------|----------------|----------|
| tiny | 39MB | ~1GB | ~32x | ★★☆☆☆ |
| base | 74MB | ~1GB | ~16x | ★★★☆☆ |
| small | 244MB | ~2GB | ~6x | ★★★★☆ |
| medium | 769MB | ~5GB | ~2x | ★★★★☆ |
| large-v3 | 2.9GB | ~10GB | 1x | ★★★★★ |

### GPU Acceleration

For faster Whisper inference, use GPU:

```bash
# Install PyTorch with CUDA
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118

# Enable fp16 in the code
result = model.transcribe(f.name, language='en', fp16=True)
```

---

## Adding Silero VAD for Voice Activity Detection

**Voice Activity Detection (VAD)** helps determine when someone is actually speaking vs. silence. This is crucial for:

- **Reducing false transcriptions** from background noise
- **Saving compute resources** by only transcribing speech segments
- **Better speaker segmentation** by detecting speech boundaries
- **Lower latency** by processing only voice segments

### Why Silero VAD?

Silero VAD is a lightweight, accurate voice activity detector that runs efficiently on CPU:

- **Fast**: Processes audio in real-time
- **Accurate**: State-of-the-art performance
- **Small**: ~2MB model size
- **Easy**: Simple Python API

### Step 1: Install Silero VAD

Add to `requirements.txt`:

```
torch
torchaudio
silero-vad
```

Or install directly:

```bash
pip install torch torchaudio
# Silero VAD is loaded from torch.hub, no separate install needed
```

### Step 2: Create VAD-Enhanced Transcription Server

Create `app_vad.py` that combines Silero VAD with Vosk or Whisper:

```python
import audioop
import base64
import json
import os
import threading
import numpy as np
import torch
import requests
from flask import Flask, request, Response
from flask_sock import Sock, ConnectionClosed
import vosk

app = Flask(__name__)
sock = Sock(app)

NEXTJS_URL = os.environ.get('NEXTJS_URL', 'http://localhost:3001')

# Load Vosk model
SCRIPT_DIR = os.path.dirname(os.path.abspath(__file__))
MODEL_PATH = os.environ.get('VOSK_MODEL_PATH', os.path.join(SCRIPT_DIR, 'model'))
print(f"Loading Vosk model from: {MODEL_PATH}")
vosk_model = vosk.Model(MODEL_PATH)
print("[OK] Vosk model loaded!")

# Load Silero VAD model
print("Loading Silero VAD model...")
vad_model, vad_utils = torch.hub.load(
    repo_or_dir='snakers4/silero-vad',
    model='silero_vad',
    force_reload=False,
    onnx=False
)
(get_speech_timestamps, save_audio, read_audio, VADIterator, collect_chunks) = vad_utils
print("[OK] Silero VAD loaded!")

browser_connections = {}
pending_transcriptions = {}
connections_lock = threading.Lock()


def send_to_browser(call_sid, data):
    """Send transcription data to the connected browser."""
    with connections_lock:
        ws = browser_connections.get(call_sid)
        if ws:
            try:
                ws.send(json.dumps({
                    'type': 'transcription',
                    'callSid': call_sid,
                    **data
                }))
                print(f"[LOG] [{data.get('speaker', '?')}]: {data.get('text', '')}")
            except Exception as e:
                print(f"Error: {e}")
        else:
            if call_sid not in pending_transcriptions:
                pending_transcriptions[call_sid] = []
            pending_transcriptions[call_sid].append(data)


class VADProcessor:
    """Process audio with Voice Activity Detection before transcription."""
    
    def __init__(self, vad_model, sample_rate=16000, threshold=0.5):
        self.vad_model = vad_model
        self.sample_rate = sample_rate
        self.threshold = threshold
        
        # Audio buffer for VAD processing
        self.audio_buffer = bytearray()
        
        # Speech buffer - only contains detected speech
        self.speech_buffer = bytearray()
        
        # VAD state
        self.is_speaking = False
        self.silence_frames = 0
        self.speech_frames = 0
        
        # Thresholds (in frames, each frame is ~32ms at 16kHz with 512 samples)
        self.min_speech_frames = 3      # Minimum speech duration to start
        self.max_silence_frames = 15    # Silence duration before ending speech
        
        # Frame size for VAD (512 samples = 32ms at 16kHz)
        self.frame_size = 512
        self.frame_bytes = self.frame_size * 2  # 16-bit audio
        
    def process_audio(self, audio_bytes):
        """
        Process audio through VAD and return speech segments.
        
        Returns:
            tuple: (speech_audio_bytes, speech_ended)
            - speech_audio_bytes: Bytes of detected speech (or None)
            - speech_ended: True if a speech segment just ended
        """
        self.audio_buffer.extend(audio_bytes)
        
        speech_ended = False
        
        # Process full frames
        while len(self.audio_buffer) >= self.frame_bytes:
            frame = bytes(self.audio_buffer[:self.frame_bytes])
            self.audio_buffer = self.audio_buffer[self.frame_bytes:]
            
            # Convert to tensor for VAD
            audio_tensor = torch.frombuffer(frame, dtype=torch.int16).float() / 32768.0
            
            # Run VAD
            speech_prob = self.vad_model(audio_tensor, self.sample_rate).item()
            is_speech = speech_prob > self.threshold
            
            if is_speech:
                self.speech_frames += 1
                self.silence_frames = 0
                
                if not self.is_speaking and self.speech_frames >= self.min_speech_frames:
                    # Speech started
                    self.is_speaking = True
                    print("Speech started")
                
                if self.is_speaking:
                    self.speech_buffer.extend(frame)
                    
            else:
                self.speech_frames = 0
                
                if self.is_speaking:
                    self.silence_frames += 1
                    self.speech_buffer.extend(frame)  # Include some silence
                    
                    if self.silence_frames >= self.max_silence_frames:
                        # Speech ended
                        self.is_speaking = False
                        speech_ended = True
                        print("🔇 Speech ended")
        
        # Return accumulated speech if segment ended
        if speech_ended and len(self.speech_buffer) > 0:
            speech_audio = bytes(self.speech_buffer)
            self.speech_buffer.clear()
            return speech_audio, True
        
        return None, False
    
    def flush(self):
        """Flush any remaining speech buffer."""
        if len(self.speech_buffer) > self.sample_rate:  # At least 0.5 sec
            speech_audio = bytes(self.speech_buffer)
            self.speech_buffer.clear()
            return speech_audio
        self.speech_buffer.clear()
        return None
    
    def reset(self):
        """Reset VAD state."""
        self.vad_model.reset_states()
        self.audio_buffer.clear()
        self.speech_buffer.clear()
        self.is_speaking = False
        self.silence_frames = 0
        self.speech_frames = 0


class VADTranscriber:
    """Combine VAD with Vosk for speech-only transcription."""
    
    def __init__(self, vosk_model, vad_model, sample_rate=16000):
        self.recognizer = vosk.KaldiRecognizer(vosk_model, sample_rate)
        self.vad = VADProcessor(vad_model, sample_rate)
        self.sample_rate = sample_rate
        
    def process_audio(self, audio_bytes):
        """Process audio through VAD, then transcribe speech segments."""
        # Run VAD
        speech_audio, speech_ended = self.vad.process_audio(audio_bytes)
        
        if speech_audio:
            # Transcribe the speech segment
            if self.recognizer.AcceptWaveform(speech_audio):
                result = json.loads(self.recognizer.Result())
                text = result.get('text', '').strip()
                if text:
                    return text
            
            # If speech ended, get final result
            if speech_ended:
                result = json.loads(self.recognizer.FinalResult())
                text = result.get('text', '').strip()
                # Reset recognizer for next utterance
                self.recognizer = vosk.KaldiRecognizer(
                    vosk_model, self.sample_rate
                )
                if text:
                    return text
        
        return None
    
    def flush(self):
        """Flush remaining audio and get final transcription."""
        speech_audio = self.vad.flush()
        if speech_audio:
            self.recognizer.AcceptWaveform(speech_audio)
        
        result = json.loads(self.recognizer.FinalResult())
        text = result.get('text', '').strip()
        self.vad.reset()
        return text if text else None


@sock.route('/transcription')
def transcription_ws(ws):
    """WebSocket endpoint for browser to receive transcriptions."""
    print("[WS] Browser connected")
    registered_call_sid = None
    
    try:
        while True:
            message = ws.receive()
            if message is None:
                break
                
            data = json.loads(message)
            
            if data.get('type') == 'ping':
                ws.send(json.dumps({'type': 'pong'}))
                
            elif data.get('type') == 'register':
                call_sid = data.get('callSid')
                if call_sid:
                    registered_call_sid = call_sid
                    with connections_lock:
                        browser_connections[call_sid] = ws
                    print(f"📱 Browser registered: {call_sid}")
                    
                    pending = pending_transcriptions.pop(call_sid, [])
                    for item in pending:
                        ws.send(json.dumps({
                            'type': 'transcription',
                            'callSid': call_sid,
                            **item
                        }))
                    
                    ws.send(json.dumps({
                        'type': 'transcription_started',
                        'callSid': call_sid
                    }))
                    
    except ConnectionClosed:
        pass
    finally:
        print("[WS] Browser disconnected")
        if registered_call_sid:
            with connections_lock:
                if browser_connections.get(registered_call_sid) == ws:
                    del browser_connections[registered_call_sid]


@sock.route('/media-stream')
def media_stream(ws):
    """Receive and transcribe Twilio media stream with VAD."""
    print("Twilio Media Stream connected (VAD + Vosk)")
    
    # Create VAD-enabled transcribers for both tracks
    inbound_transcriber = VADTranscriber(vosk_model, vad_model)
    outbound_transcriber = VADTranscriber(vosk_model, vad_model)
    
    call_sid = None
    
    try:
        while True:
            message = ws.receive()
            if message is None:
                break
                
            packet = json.loads(message)
            event = packet.get('event')
            
            if event == 'start':
                call_sid = packet.get('start', {}).get('callSid')
                print(f"🎙️ Stream started: {call_sid}")
                
            elif event == 'media':
                media = packet.get('media', {})
                track = media.get('track', 'inbound')
                payload = media.get('payload', '')
                
                if not payload:
                    continue
                
                # Decode and convert audio
                try:
                    audio = base64.b64decode(payload)
                    audio = audioop.ulaw2lin(audio, 2)
                    audio = audioop.ratecv(audio, 2, 1, 8000, 16000, None)[0]
                except:
                    continue
                
                # Choose transcriber
                if track == 'outbound':
                    transcriber = outbound_transcriber
                    speaker = 'agent'
                else:
                    transcriber = inbound_transcriber
                    speaker = 'customer'
                
                # Process with VAD + transcription
                text = transcriber.process_audio(audio)
                if text and call_sid:
                    send_to_browser(call_sid, {
                        'text': text,
                        'speaker': speaker,
                        'isFinal': True,
                        'timestamp': media.get('timestamp', '')
                    })
                        
            elif event == 'stop':
                print(f"Stream stopped: {call_sid}")
                
                # Flush remaining audio
                for transcriber, speaker in [
                    (inbound_transcriber, 'customer'),
                    (outbound_transcriber, 'agent')
                ]:
                    text = transcriber.flush()
                    if text and call_sid:
                        send_to_browser(call_sid, {
                            'text': text,
                            'speaker': speaker,
                            'isFinal': True,
                            'timestamp': ''
                        })
                
                with connections_lock:
                    browser_ws = browser_connections.get(call_sid)
                    if browser_ws:
                        try:
                            browser_ws.send(json.dumps({
                                'type': 'transcription_ended',
                                'callSid': call_sid
                            }))
                        except:
                            pass
                break
                
    except ConnectionClosed:
        print("Media Stream closed")
    except Exception as e:
        print(f"[ERROR] Error: {e}")


# Reverse proxy (same as before)
@app.route('/', defaults={'path': ''}, methods=['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS'])
@app.route('/<path:path>', methods=['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS'])
def proxy(path):
    if path in ['transcription', 'media-stream', 'health']:
        return {'error': 'Use WebSocket'}, 400
    
    target_url = f"{NEXTJS_URL}/{path}"
    if request.query_string:
        target_url += f"?{request.query_string.decode()}"
    
    try:
        headers = {k: v for k, v in request.headers if k.lower() not in [
            'host', 'connection', 'transfer-encoding', 'upgrade'
        ]}
        resp = requests.request(
            method=request.method,
            url=target_url,
            headers=headers,
            data=request.get_data(),
            cookies=request.cookies,
            allow_redirects=False,
            stream=True,
            timeout=60
        )
        excluded = ['content-encoding', 'content-length', 'transfer-encoding', 'connection']
        response_headers = [(n, v) for n, v in resp.raw.headers.items() if n.lower() not in excluded]
        return Response(resp.content, status=resp.status_code, headers=response_headers)
    except requests.exceptions.ConnectionError:
        return {'error': 'Next.js not running'}, 502


if __name__ == '__main__':
    port = int(os.environ.get('PORT', 3000))
    print(f"\nVAD + Vosk Transcription Server on port {port}\n")
    app.run(host='0.0.0.0', port=port, debug=False)
```

### Step 3: Understanding the VAD Flow

```
Audio Stream
     |
     v
+-------------------------------------------------------------------+
|                         VAD Processor                             |
|                                                                   |
|   +-----------+    +----------------+    +---------------+        |
|   | Buffer    |--->| Silero VAD     |--->| Speech        |        |
|   | Audio     |    | (per frame)    |    | Detection     |        |
|   +-----------+    +----------------+    +---------------+        |
|                                                 |                 |
|                                                 v                 |
|                          +-----------------------------+          |
|                          | Is Speech Probability >     |          |
|                          | Threshold (0.5)?            |          |
|                          +-----------------------------+          |
|                               |               |                   |
|                           YES |               | NO                |
|                               v               v                   |
|                       +------------+    +----------------+        |
|                       | Add to     |    | Count          |        |
|                       | Speech     |    | Silence        |        |
|                       | Buffer     |    | Frames         |        |
|                       +------------+    +----------------+        |
|                               |               |                   |
|                               |               v                   |
|                               |    +--------------------+         |
|                               |    | Silence > 15       |         |
|                               |    | frames? Speech     |         |
|                               |    | Ended!             |         |
|                               |    +--------------------+         |
|                               |               |                   |
|                               v               v                   |
|                       +-------------------------------+           |
|                       |    Return Speech Segment      |           |
|                       +-------------------------------+           |
|                                       |                           |
+---------------------------------------+---------------------------+
                                        |
                                        v
                             +----------------------+
                             |  Vosk/Whisper        |
                             |  Transcription       |
                             +----------------------+
                                        |
                                        v
                               [ Transcribed Text ]
```

### Step 4: Combining VAD with Whisper

For best accuracy, combine Silero VAD with Whisper:

```python
class VADWhisperTranscriber:
    """Combine VAD with Whisper for high-accuracy speech transcription."""
    
    def __init__(self, whisper_model, vad_model, sample_rate=16000):
        self.whisper_model = whisper_model
        self.vad = VADProcessor(vad_model, sample_rate)
        self.sample_rate = sample_rate
        
    def process_audio(self, audio_bytes):
        """Process audio through VAD, then transcribe with Whisper."""
        speech_audio, speech_ended = self.vad.process_audio(audio_bytes)
        
        if speech_ended and speech_audio and len(speech_audio) > self.sample_rate:
            # Save speech segment to temp file
            with tempfile.NamedTemporaryFile(suffix='.wav', delete=False) as f:
                with wave.open(f.name, 'wb') as wav:
                    wav.setnchannels(1)
                    wav.setsampwidth(2)
                    wav.setframerate(self.sample_rate)
                    wav.writeframes(speech_audio)
                
                # Transcribe with Whisper
                result = self.whisper_model.transcribe(
                    f.name,
                    language='en',
                    fp16=False
                )
                os.unlink(f.name)
                
                return result.get('text', '').strip()
        
        return None
```

### VAD Configuration Tips

```python
# More sensitive (catches softer speech, but more false positives)
vad = VADProcessor(vad_model, threshold=0.3)

# Less sensitive (misses quiet speech, but fewer false positives)
vad = VADProcessor(vad_model, threshold=0.7)

# Quicker speech end detection (lower latency)
vad.max_silence_frames = 8  # ~256ms of silence ends speech

# Slower speech end detection (captures natural pauses)
vad.max_silence_frames = 25  # ~800ms of silence ends speech
```

### Benefits of VAD

| Without VAD | With VAD |
|-------------|----------|
| Processes all audio | Only processes speech |
| Background noise triggers transcription | Ignores background noise |
| CPU always busy | CPU idle during silence |
| Random garbage text from noise | Clean, accurate transcriptions |
| No clear utterance boundaries | Clear start/end of each utterance |

---

## Comparison: Vosk vs Whisper vs VAD+Vosk vs VAD+Whisper

| Setup | Speed | Accuracy | Latency | Best For |
|-------|-------|----------|---------|----------|
| **Vosk** | ⚡⚡⚡ | ★★★☆☆ | ~200ms | Real-time, resource-constrained |
| **Whisper (base)** | ⚡⚡ | ★★★★☆ | ~2-3s | Better accuracy, offline |
| **Whisper (large)** | ⚡ | ★★★★★ | ~5-10s | Maximum accuracy |
| **VAD + Vosk** | ⚡⚡⚡ | ★★★★☆ | ~300ms | Real-time, clean output |
| **VAD + Whisper** | ⚡⚡ | ★★★★★ | ~2-4s | Best accuracy + clean output |

### Recommendation

- **For real-time display**: Use VAD + Vosk
- **For accuracy**: Use VAD + Whisper (medium or large)
- **For cost-sensitive deployments**: Use Vosk with larger model
- **For production call centers**: Use VAD + Whisper with GPU

---

## Conclusion

Congratulations! You've built a complete browser-based phone system with real-time speech transcription. This architecture gives you:

- **Full control** over your transcription pipeline
- **Privacy** with on-premise speech processing
- **Cost savings** by avoiding per-minute transcription fees
- **Low latency** transcription for real-time applications

The combination of Twilio's reliable telephony infrastructure with Vosk's offline speech recognition creates a powerful foundation for building call center applications, accessibility tools, or any system requiring real-time speech-to-text.

### Resources

- [Twilio Voice SDK Documentation](https://www.twilio.com/docs/voice/sdks/javascript)
- [Vosk Speech Recognition](https://alphacephei.com/vosk/)
- [Twilio Media Streams](https://www.twilio.com/docs/voice/media-streams)
- [Next.js Documentation](https://nextjs.org/docs)

---

*Have questions or feedback? Feel free to reach out or leave a comment below!*

---

**Tags**: Twilio, Voice, Speech Recognition, Vosk, Next.js, Python, WebSocket, Real-time, Transcription, Tutorial
