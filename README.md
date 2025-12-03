# VideoCash — Watch & Earn React + Firebase SPA

A single-page React application where users can watch videos to earn points, integrated with **Google Login**, **real-time Firestore updates**, and **AdGem HMAC postback validation**.

## Features

- Login with **Google** using Firebase Auth
- Real-time point updates with **Firestore `onSnapshot()`**
- Watch videos and earn points
- Request payouts (points reset to 0 after request)
- **AdGem HMAC postback placeholder** for secure offerwall rewards
- Fallback **in-memory database** if Firestore is unavailable (for testing)
- Responsive and simple UI

## Installation

1. Clone the repository:
```bash
git clone https://github.com/YOUR_USERNAME/videocash.git
cd videocash
```

2. Install dependencies:
```bash
npm install
```

3. Configure Firebase:
- Replace the `firebaseConfig` object in `App.jsx` with your Firebase project details.
- Enable **Google Authentication** in Firebase Auth.
- Enable **Firestore** in your Firebase project.

## Running the App

```bash
npm start
```

This will start the app at `http://localhost:3000`.

## Usage

1. Click **Login with Google** to sign in.
2. Watch videos from the list.
3. Points will update in real-time.
4. Click **Request Payout** to simulate a payout (points reset to 0).

## AdGem Integration

- The app contains a placeholder function `handleAdGemPostback(uid, points, signature)` to simulate server-side HMAC validation.
- In production, verify the HMAC signature received from AdGem on your backend before crediting the user.

## Code Structure

- Single-file SPA (`App.jsx`) for demonstration purposes.
- `FIRE` API handles Firestore interactions with in-memory fallback.
- Reward system calculates points per video duration.

## Notes

- Currently uses a simple UI for demonstration.
- For production, separate components, backend validation, and secure offerwall endpoints are recommended.

## License

MIT
