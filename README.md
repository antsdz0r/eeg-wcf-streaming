# NeuroStream

A WCF client–server pipeline that streams EEG recordings sample by sample, validates every row, stores clean data per participant and session, and raises real-time alerts for stress spikes, relaxation drops, poor electrode contact, low battery and timestamp gaps.

A university project for the Virtualization course (Virtuelizacija procesa), built on .NET Framework 4.7.2.

---

## What it does

The **client** scans a folder for EEG exports named `subject_<id>_results.csv`, parses them, and pushes each row to the **service** over `net.tcp`. The service checks each sample, writes accepted rows to disk, and fires C# events that the host console subscribes to.

```
EEG/subject_7_results.csv
        │  CsvParser
        ▼
   Client ── StartSession(meta) ──▶ EegService ── creates Data/7/<date>/session.csv
          ── PushSample(row) ×N ──▶            ── validate → append → analytics → events
          ── EndSession() ────────▶            ── flush & close files
```

### Data format

Each CSV row has 16 columns:

| Group | Columns |
|---|---|
| Time | `Timestamp` |
| EEG channels | `AF3`, `T7`, `Pz`, `T8`, `AF4` |
| Cognitive metrics | `Attention`, `Engagement`, `Excitement`, `Interest`, `Relaxation`, `Stress` |
| Device and stimulus | `Battery`, `ContactQuality`, `SlideIndex`, `SetIndex` |

The participant ID is taken from the file name: `subject_7_results.csv` → `7`.

## Service contract

```csharp
[ServiceContract]
public interface IEegService
{
    AckResponse StartSession(EegMeta meta);   // ValidationFault
    AckResponse PushSample(EegSample sample); // DataFormatFault, ValidationFault
    AckResponse EndSession();
}
```

Every call returns an `AckResponse` with `IsAck`, a message, and a `SessionStatus` of `InProgress` or `Completed`.

## Validation

A sample is **rejected** with a typed WCF fault and logged to `rejects.csv` if:

- it is null (`DataFormatFault`)
- its `RowIndex` is not greater than the previous one (`ValidationFault`)
- its timestamp is missing or invalid (`DataFormatFault`)
- any cognitive metric is negative (`ValidationFault`)
- any EEG channel is `NaN` or infinite (`DataFormatFault`)

On the client side, rows that can't be parsed at all are skipped and written to `client_errors.log`.

## Analytics and events

Accepted samples are checked against thresholds from `Service/App.config`:

| Event | Fires when | Default |
|---|---|---|
| `OnStressSpike` | Stress changes by more than the threshold between consecutive rows (up or down) | 10.0 |
| `OnRelaxationDrop` | Relaxation falls by more than the threshold | 10.0 |
| `OnPoorContact` | `ContactQuality` is below the minimum | 3 |
| `OnLowBattery` | `Battery` is below the threshold | 20 % |
| `OnTimeSkew` | The gap between consecutive timestamps exceeds the limit | 2000 ms |

The service also raises `OnTransferStarted`, `OnSampleReceived`, `OnTransferCompleted` and a catch-all `OnWarningRaised`. The host in `Service/Program.cs` subscribes to all of them and prints them to the console.

## Output

```
Data/
└── <participantId>/
    └── <yyyy-MM-dd>/
        ├── session.csv   accepted samples (+ RowIndex column)
        └── rejects.csv   rejected rows and device warnings, with reason
```

Files open in append mode, so several sessions on the same day go into the same folder.

## Project structure

```
EEG_WCF.sln
├── Common/    Service contract, data contracts (EegSample, EegMeta, AckResponse),
│              fault contracts, enums, event-args classes
├── Service/   EegService implementation and the console host
└── Client/    CsvParser and the console client that streams files
```

## Running it

**Requirements:** Windows and Visual Studio with .NET Framework 4.7.2 targeting support.

1. Open `EEG_WCF.sln` and build the solution.
2. Put your EEG exports in a folder named `EEG` next to the client executable, or set `EegDataPath` in `Client/App.config`. Subfolders are searched too.
3. Start **Service**. It listens on `net.tcp://localhost:5000/EegService`.
4. Start **Client**. It streams every matching file and prints an acknowledgement for each session.

If port 5000 is taken, change it in both `App.config` files and in `Client/Program.cs`, where the endpoint is also hard-coded.

## Known gaps

- The client builds its binding and endpoint in code, so the `<client>` section of `Client/App.config` isn't used.
- `ChannelOutOfBandPct` and `ClientOutputPath` are defined in config but not used yet. `Client/Downloaded/` is a placeholder for the same unfinished feature.
- The service is a single instance (`InstanceContextMode.Single`) with one set of session fields, so it handles one client at a time.
