# Project Plan: Custom Jellyfin App (Firestick + Android Phones)

## Overview

Build a custom Netflix-style front-end for your Jellyfin media server using **React Native**, targeting two platforms from a single codebase:

- **Amazon Firestick** (Fire TV) — big-screen TV layout with D-pad/remote navigation
- **Samsung Galaxy phones** — mobile layout with touch navigation

The app talks to Jellyfin's REST API for all media data. You package and sideload an APK onto the Firestick, and install directly on Android phones.

**Estimated total time:** 3–5 weekends (iterative — core playback works early, polish takes longer)  
**Difficulty:** Intermediate–Advanced (React Native experience helpful but not required)

---

## Architecture Overview

```
Jellyfin Server (home network / Tailscale)
        ↓  REST API
React Native App
        ├── Phone layout  (touch, portrait, scrolling grid)
        └── TV layout     (D-pad, landscape, row-based browsing)
        ↓
Two APK builds
        ├── phone.apk  → sideload to Samsung phones
        └── tv.apk     → sideload to Firestick
```

---

## What You'll Need

- A development machine (Mac or Windows laptop) with Node.js installed
- Android Studio (free) — for building APKs
- A USB cable or ADB over WiFi — to sideload onto the Firestick
- Your Jellyfin server running and accessible (local or via Tailscale)
- React Native knowledge (or willingness to learn — the official docs are good)

---

## Phase 1 — Development Environment

- [ ] Install **Node.js** (LTS version) from nodejs.org
- [ ] Install **Android Studio** from developer.android.com
  - During setup, install Android SDK, Android SDK Platform, and Android Virtual Device
- [ ] Install the React Native CLI:
  ```bash
  npm install -g @react-native-community/cli
  ```
- [ ] Create the project using the React Native TV template:
  ```bash
  npx react-native init JellyPlayer --template @react-native-tvos/react-native-template-typescript-tv
  ```
  > `react-native-tvos` is a maintained fork of React Native that adds TV/D-pad support while remaining fully compatible with phone builds — one codebase, both targets.
- [ ] Confirm it builds for Android:
  ```bash
  cd JellyPlayer
  npx react-native run-android
  ```

---

## Phase 2 — Jellyfin API Integration

Create a service layer that wraps all Jellyfin API calls.

- [ ] Install axios for HTTP requests:
  ```bash
  npm install axios
  ```
- [ ] Create `src/services/jellyfin.ts`:

```typescript
import axios from 'axios';

const SERVER_URL = 'http://192.168.1.100:8096'; // or Tailscale IP for remote
let AUTH_TOKEN = '';
let USER_ID = '';

const api = axios.create({ baseURL: SERVER_URL });

// Authenticate and store token
export async function login(username: string, password: string) {
  const response = await api.post('/Users/AuthenticateByName', {
    Username: username,
    Pw: password,
  }, {
    headers: {
      'X-Emby-Authorization': `MediaBrowser Client="JellyPlayer", Device="App", DeviceId="jellyplayer-001", Version="1.0.0"`,
    },
  });
  AUTH_TOKEN = response.data.AccessToken;
  USER_ID = response.data.User.Id;
  return response.data;
}

// Get all films
export async function getFilms() {
  return api.get(`/Users/${USER_ID}/Items`, {
    params: { IncludeItemTypes: 'Movie', Recursive: true, SortBy: 'SortName', Fields: 'Overview,Genres' },
    headers: { 'X-Emby-Token': AUTH_TOKEN },
  });
}

// Get all TV shows
export async function getTVShows() {
  return api.get(`/Users/${USER_ID}/Items`, {
    params: { IncludeItemTypes: 'Series', Recursive: true, SortBy: 'SortName' },
    headers: { 'X-Emby-Token': AUTH_TOKEN },
  });
}

// Get continue watching
export async function getContinueWatching() {
  return api.get(`/Users/${USER_ID}/Items/Resume`, {
    params: { MediaTypes: 'Video', Limit: 10 },
    headers: { 'X-Emby-Token': AUTH_TOKEN },
  });
}

// Get next episode for a series
export async function getNextUp() {
  return api.get(`/Shows/NextUp`, {
    params: { UserId: USER_ID, Limit: 10 },
    headers: { 'X-Emby-Token': AUTH_TOKEN },
  });
}

// Get artwork URL
export function getImageUrl(itemId: string, type: 'Primary' | 'Backdrop' | 'Thumb' = 'Primary') {
  return `${SERVER_URL}/Items/${itemId}/Images/${type}?maxWidth=400`;
}

// Get direct stream URL
export function getStreamUrl(itemId: string) {
  return `${SERVER_URL}/Videos/${itemId}/stream?static=true&api_key=${AUTH_TOKEN}`;
}

// Report playback progress (so Continue Watching works)
export async function reportProgress(itemId: string, positionTicks: number) {
  return api.post(`/Sessions/Playing/Progress`, {
    ItemId: itemId,
    PositionTicks: positionTicks,
  }, {
    headers: { 'X-Emby-Token': AUTH_TOKEN },
  });
}

// Mark as stopped
export async function reportStopped(itemId: string, positionTicks: number) {
  return api.post(`/Sessions/Playing/Stopped`, {
    ItemId: itemId,
    PositionTicks: positionTicks,
  }, {
    headers: { 'X-Emby-Token': AUTH_TOKEN },
  });
}
```

- [ ] Test each function from a simple test screen to confirm data returns correctly

---

## Phase 3 — Platform Detection & Layout Strategy

The app uses the same screens but switches layout based on whether it's running on a TV or phone.

- [ ] Install the TV detection helper:
  ```bash
  npm install react-native-device-info
  ```
- [ ] Create `src/utils/platform.ts`:
  ```typescript
  import { Platform } from 'react-native';
  import DeviceInfo from 'react-native-device-info';

  export const isTV = Platform.isTV || DeviceInfo.getDeviceType() === 'Tv';
  ```
- [ ] Use this throughout the app to conditionally apply TV or phone layouts:
  ```typescript
  import { isTV } from '../utils/platform';

  const styles = StyleSheet.create({
    container: {
      flexDirection: isTV ? 'row' : 'column',
      paddingHorizontal: isTV ? 60 : 16,
    }
  });
  ```

---

## Phase 4 — Core Screens

### 4a — Login Screen

- [ ] Create `src/screens/LoginScreen.tsx`:
  - Simple username/password form
  - Calls `login()` on submit
  - Stores credentials securely using `react-native-keychain`
  - Auto-logs in on subsequent app opens if token is stored
  ```bash
  npm install react-native-keychain
  ```

### 4b — Home Screen

The main browsing screen. Rows of content Netflix-style.

- [ ] Create `src/screens/HomeScreen.tsx` with these rows:
  - Continue Watching
  - Next Up (next episodes of shows you're watching)
  - Recently Added Films
  - Recently Added TV
  - All Films (horizontal scroll)
  - All TV Shows (horizontal scroll)
- [ ] Each row is a horizontal `FlatList` of poster cards
- [ ] Tapping a card opens the Detail Screen

### 4c — Detail Screen

- [ ] Create `src/screens/DetailScreen.tsx`:
  - Large backdrop image at top
  - Title, year, genre, synopsis
  - **Play** button (primary action)
  - For TV shows: episode list below
  - Resume position shown if partially watched ("Resume from 42:10")

### 4d — Player Screen

- [ ] Install the video player:
  ```bash
  npm install react-native-video
  ```
- [ ] Create `src/screens/PlayerScreen.tsx`:
  ```typescript
  import Video from 'react-native-video';

  // Key behaviours:
  // - Full screen, landscape locked
  // - Report progress to Jellyfin every 10 seconds (so Continue Watching stays accurate)
  // - Show/hide controls on tap (phone) or D-pad OK press (TV)
  // - Auto-play next episode with 10-second countdown overlay
  // - Report stopped with final position on exit
  ```
- [ ] Implement progress reporting on an interval:
  ```typescript
  useEffect(() => {
    const interval = setInterval(() => {
      if (currentTime > 0) {
        reportProgress(itemId, Math.floor(currentTime * 10_000_000)); // Jellyfin uses ticks
      }
    }, 10_000);
    return () => clearInterval(interval);
  }, [currentTime]);
  ```

---

## Phase 5 — TV-Specific: D-pad Navigation

This is the most Firestick-specific work. The remote's D-pad moves focus between items; pressing OK selects. React Native TV handles most of this automatically but you need to configure it.

- [ ] Ensure all interactive elements use `TouchableHighlight` (not `TouchableOpacity`) — it shows focus state on TV
- [ ] Add focus styles so the selected item is visually obvious:
  ```typescript
  const [focused, setFocused] = useState(false);

  <TouchableHighlight
    onFocus={() => setFocused(true)}
    onBlur={() => setFocused(false)}
    style={[styles.card, focused && styles.cardFocused]}
  >
  ```
  ```typescript
  cardFocused: {
    transform: [{ scale: 1.08 }],
    borderWidth: 3,
    borderColor: '#FFFFFF',
  }
  ```
- [ ] Use `TVFocusGuideView` to control where focus moves at the edges of rows:
  ```typescript
  import { TVFocusGuideView } from 'react-native';

  <TVFocusGuideView destinations={[nextRowRef]}>
    <FlatList ... />
  </TVFocusGuideView>
  ```
- [ ] Handle the back button on the remote:
  ```typescript
  import { BackHandler } from 'react-native';

  useEffect(() => {
    const sub = BackHandler.addEventListener('hardwareBackPress', () => {
      navigation.goBack();
      return true;
    });
    return () => sub.remove();
  }, []);
  ```
- [ ] Test thoroughly with the Firestick remote — D-pad navigation feels very different to touch and needs iteration

---

## Phase 6 — Phone-Specific Polish

- [ ] Lock the player to landscape using `react-native-orientation-locker`
- [ ] Add swipe-to-go-back on the detail screen
- [ ] Ensure touch targets are large enough (minimum 48px)
- [ ] Add a bottom tab bar for Home / Films / TV / Search
- [ ] Test on both Shell's and your own phones

---

## Phase 7 — Build & Deploy to Phones

- [ ] Generate a signing keystore (one-time setup):
  ```bash
  keytool -genkeypair -v -storetype PKCS12 -keystore jellyplayer.keystore \
    -alias jellyplayer -keyalg RSA -keysize 2048 -validity 10000
  ```
- [ ] Configure signing in `android/app/build.gradle`
- [ ] Build a release APK:
  ```bash
  cd android
  ./gradlew assembleRelease
  ```
  Output: `android/app/build/outputs/apk/release/app-release.apk`
- [ ] Transfer the APK to both phones (AirDrop equivalent: send via Google Messages, email, or copy to shared folder)
- [ ] On each Samsung phone:
  - Settings → Install unknown apps → allow for your file manager/browser
  - Open the APK and install
- [ ] Test on both phones end-to-end

---

## Phase 8 — Build & Sideload to Firestick

Fire OS is Android under the hood — you sideload the same way, via ADB.

- [ ] **Enable developer mode on the Firestick:**
  - Settings → My Fire TV → About → click "Fire TV Stick" 7 times rapidly
  - Settings → My Fire TV → Developer Options → ADB Debugging: ON
  - Settings → My Fire TV → Developer Options → Apps from Unknown Sources: ON

- [ ] **Connect via ADB over WiFi** (no USB needed for Firestick):
  - Find the Firestick's IP: Settings → My Fire TV → About → Network → IP Address
  - On your development machine:
    ```bash
    adb connect 192.168.1.XXX:5555
    adb devices  # Confirm it appears
    ```

- [ ] **Build the TV APK:**
  ```bash
  # In android/app/build.gradle, ensure TV support is included
  # Then build:
  ./gradlew assembleRelease
  ```

- [ ] **Install to Firestick:**
  ```bash
  adb install android/app/build/outputs/apk/release/app-release.apk
  ```

- [ ] The app will appear under **Apps → Your Apps & Channels → See All** on the Firestick home screen
- [ ] Pin it to the home screen for easy access

---

## Phase 9 — Netflix-Feel Polish

Once the core is working, these are what lift it from "functional" to "feels right":

- [ ] **Autoplay next episode** — 10-second countdown overlay at the end of an episode with a "Play Next" button and a cancel option
- [ ] **Skip intro** — Jellyfin detects intro timestamps; add a "Skip Intro" button that jumps past it
- [ ] **Splash screen** — custom logo on launch instead of white flash
- [ ] **Smooth image loading** — use `react-native-fast-image` for snappy poster loads with shimmer placeholders
- [ ] **Offline indicator** — detect when Tailscale/server is unreachable and show a friendly message rather than a crash
- [ ] **Kids profile** — a separate profile for Ed that only shows content from a designated kids library in Jellyfin

---

## Maintenance

- When you add new media to Jellyfin it appears in the app automatically — no app update needed
- If you change your server IP or Tailscale address, update `SERVER_URL` in `jellyfin.ts` and rebuild
- Keep `react-native-tvos` updated — it tracks the main React Native release cadence

---

## Useful Resources

- [React Native TV (react-native-tvos)](https://github.com/react-native-tvos/react-native-tvos)
- [Jellyfin API Reference](https://api.jellyfin.org)
- [react-native-video](https://github.com/TheWidlarzGroup/react-native-video)
- [ADB Sideloading to Firestick](https://www.aftvnews.com/how-to-install-an-android-apk-on-the-amazon-fire-tv-or-fire-tv-stick-using-adb/)
- [React Native Signed APK guide](https://reactnative.dev/docs/signed-apk-android)

---

## Platform Comparison

| Feature | Phone | Firestick |
|---|---|---|
| Navigation | Touch | D-pad remote |
| Layout | Portrait grid, tab bar | Landscape rows, no tab bar |
| Install method | APK sideload | ADB over WiFi |
| Orientation | Auto (portrait/landscape) | Landscape locked |
| Back button | Swipe / button | Remote back button |
| Focus states | Not needed | Essential |
