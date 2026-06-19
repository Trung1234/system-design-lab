# edu_mul_agents: Blazor Server AI Assistant with Real-time Voice

`edu_mul_agents` is a Blazor Server application designed to provide an interactive AI Assistant with real-time voice capabilities, leveraging Microsoft Entra ID for authentication and Microsoft Graph API for education data. This project acts as a **v1 prototype** to validate architectural patterns and workflow.

## Table of Contents
- [Project Overview](#project-overview)
- [Essential Commands](#essential-commands)
- [Architecture: Clean Architecture (3-Layer)](#architecture-clean-architecture-3-layer)
- [Real-Time Voice Architecture](#real-time-voice-architecture)
- [Optimizing STT -> LLM -> TTS Pipeline](#optimizing-stt---llm---tts-pipeline)
- [Deployment Notes](#deployment-notes)
- [Version and Constraints](#version-and-constraints)
- [AI Assistant Definition](#ai-assistant-definition)

## Project Overview

- **Framework**: .NET 8 (LTS)
- **UI**: Blazor Server with SignalR for real-time updates
- **Authentication**: Microsoft Entra ID with Microsoft.Identity.Web
- **Data Source**: Microsoft Graph API (Education endpoints)
- **Voice**: Azure nhà cung cấp dịch vụ AI Realtime API (voice-to-voice conversation)
- **Deployment**: Azure App Service

## Essential Commands

### Build and Run
```bash
# Build solution
dotnet build edu_mul_agents.sln

# Run the Web project
dotnet run --project src/edu_mul_agents.Web/edu_mul_agents.Web.csproj

# Build in Release mode
dotnet build edu_mul_agents.sln -c Release
```

### Project Structure
```bash
# Restore dependencies for all projects
dotnet restore

# Clean build artifacts
dotnet clean
```

## Architecture: Clean Architecture (3-Layer)

The project strictly follows Clean Architecture principles with a clear separation of concerns across three main layers: Web, Application Core, and Infrastructure.

| Layer | Project | Responsibilities | Dependencies |
|-------|---------|------------------|--------------|
| **Web** | `edu_mul_agents.Web` | UI (Blazor components), SignalR Hubs, Auth Flow | → ApplicationCore, Infrastructure |
| **Application Core** | `edu_mul_agents.ApplicationCore` | Business logic, Interfaces, Domain Models, MockData | → None (zero dependencies) |
| **Infrastructure** | `edu_mul_agents.Infrastructure` | External API clients, Microsoft Graph SDK, Service implementations | → ApplicationCore only |

**Critical Dependency Rules:**
- **Application Core MUST NOT** reference any other project or external libraries.
- **Infrastructure** implements interfaces defined in Application Core.
- **Web** depends on both Application Core and Infrastructure.
- All service interfaces are defined in Application Core (`/Interfaces`).
- All domain models live in Application Core (`/Models`).

## Real-Time Voice Architecture

The application integrates real-time voice capabilities using Azure nhà cung cấp dịch vụ AI.

### Components

| Component | Location | Purpose |
|-----------|----------|---------|
| `IRealtimeVoiceService` | ApplicationCore/Interfaces | Service interface |
| `RealtimeVoiceService` | Infrastructure/Services | WebSocket client to Azure nhà cung cấp dịch vụ AI |
| `voiceRealtime.js` | Web/wwwroot/js | Browser audio capture/playback (Web Audio API) |
| `VoiceMode.razor` | Web/Components/Pages | JS Interop bridge |

### Data Flow

```mermaid
graph TD
    A[Browser (getUserMedia)] --> B(JS Interop);
    B --> C(RealtimeVoiceService C#);
    C --> D(Azure nhà cung cấp dịch vụ AI Realtime - STT/LLM/TTS);
    D --> C;
    C --> B;
    B --> E[Browser (AudioContext)];
```

### Configuration

Voice settings are managed in `appsettings.json` under `AzureOpenAIRealtime`.

## Optimizing STT -> LLM -> TTS Pipeline

The real-time voice interaction involves a Speech-to-Text (STT) -> Large Language Model (LLM) -> Text-to-Speech (TTS) pipeline. Optimizing this pipeline is crucial for a fluid conversational experience.

### Current Implementation Flow

1. **Speech-to-Text (STT)**: User's speech is captured in the browser, sent via WebSockets to `RealtimeVoiceService`, and then to Azure nhà cung cấp dịch vụ AI for STT conversion.
2. **Large Language Model (LLM)**: The transcribed text from STT is fed into Azure nhà cung cấp dịch vụ AI's LLM for processing and generating a response.
3. **Text-to-Speech (TTS)**: The LLM's text response is then converted into audio by Azure nhà cung cấp dịch vụ AI's TTS service and streamed back to the browser for playback.

### Optimization Strategy

To minimize latency and improve responsiveness, several strategies can be employed:

1.  **Streamed STT & TTS**: Instead of waiting for the full speech input before sending to LLM, or the full LLM response before starting TTS, stream audio chunks.
    *   **STT**: Send small audio chunks to the STT service as they are captured. Modern STT services can provide partial transcripts which can be immediately sent to the LLM.
    *   **TTS**: Once the LLM starts generating its response, feed the text to the TTS service in chunks. As TTS audio chunks become available, stream them back to the client for immediate playback. This creates an "overlapping" effect where the user hears the beginning of the response while the end is still being generated.

2.  **Early LLM Processing with Partial STT**: When the STT service provides partial results, the LLM can begin processing them. This allows the LLM to "think" while the user is still speaking or while more accurate STT results are being generated.

3.  **Client-side Caching/Prediction**: For common phrases or predictable responses, client-side caching of TTS audio or predictive text generation could reduce server round-trips. (Less applicable for a general AI Assistant, but valuable for specific interactive elements).

4.  **Optimized WebSocket Communication**: Ensure efficient data serialization and minimal overhead in WebSocket frames to reduce transmission latency.

### Optimized Pipeline Diagram

```mermaid
graph TD
    subgraph User Interaction
        A[User Speaks] --> B(Browser Audio Capture);
    end

    subgraph Client-side Processing
        B --> C{Chunk Audio};
        C -- Chunk 1 --> D[JS Interop (RealtimeVoiceService)];
        C -- Chunk N --> D;
    end

    subgraph Server-side Processing (Azure nhà cung cấp dịch vụ AI Realtime)
        D --> E[Streamed STT (Partial Text)];
        E -- Partial Text --> F[LLM Processing (Early Start)];
        E -- Final Text --> F;
        F -- Partial Response --> G[Streamed TTS (Partial Audio)];
        F -- Final Response --> G;
    end

    subgraph Client-side Playback
        G -- Partial Audio Stream --> H[JS Interop (Browser Audio)];
        G -- Final Audio Stream --> H;
        H --> I[User Hears Response];
    end

    style C fill:#f9f,stroke:#333,stroke-width:2px
    style D fill:#bbf,stroke:#333,stroke-width:2px
    style E fill:#fcf,stroke:#333,stroke-width:2px
    style F fill:#bfb,stroke:#333,stroke-width:2px
    style G fill:#fcc,stroke:#333,stroke-width:2px
    style H fill:#bbf,stroke:#333,stroke-width:2px
```
This optimized pipeline aims to reduce perceived latency by overlapping processing steps and streaming data as soon as it's available.

## Deployment Notes

Refer to the `CLAUDE.md` for detailed deployment instructions, including Azure App Service requirements and configuration settings.

## Version and Constraints

- **Current Version**: v1 (prototype)
- **Constraint**: Microsoft technologies only.
- **No Database**: Uses JSON files in `ApplicationCore/MockData/`.
- **No Tests**: Test project not created yet.

## AI Assistant Definition

The "AI Assistant" in this codebase is a server-side component processing user requests, executing logic in Application Core and Infrastructure, and returning results in real-time via SignalR or JS Interop. It is NOT an autonomous system or a separate process.