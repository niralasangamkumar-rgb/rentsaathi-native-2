# Quick Test Guide - Publish Listing Fix

## What Was Fixed

| Issue | Root Cause | Fix |
|-------|-----------|-----|
| Empty "No listings yet" after publish | Navigation to wrong route `/(owner)/my-listings` | Changed to `/(tabs)/my-listings` with `router.replace()` |
| No visual feedback during load | Missing loading state | Added `ActivityIndicator` with spinner |
| Route confusion | Duplicate `app/my-listings.tsx` | Converted to redirect component |

## How to Test

### 1. Publish a New Listing
```
1. Login as owner
2. Go to Add Listing
3. Fill form (title, rent, city, locality, phone)
4. Add at least 1 image
5. Click "Publish"
```

### 2. Watch Console Logs
Should see in order:
```
[AddListing] Publishing NEW listing with currentUser.uid: [YOUR_UID]
[AddListing] Firestore data with ownerId: [YOUR_UID]
[AddListing] Listing saved to Firestore with ownerId: [YOUR_UID]
[AddListing] Navigating to /(tabs)/my-listings

[MyListings] Screen focused, refreshing listings...
[MyListings] Current user ID: [YOUR_UID]
[ListingContext] Loaded X listings from Firestore
[MyListings] Listing [ID] (ownerId: [YOUR_UID]) matches user [YOUR_UID]: true
[MyListings] Filtered owner listings: 1
```

### 3. Verify Navigation & UI
- [ ] Success alert appears
- [ ] Click OK → navigates to My Listings
- [ ] Loading spinner appears for ~500ms
- [ ] No "No listings yet" message
- [ ] Your new listing appears with:
  - [ ] Correct title
  - [ ] Correct price
  - [ ] Correct location
  - [ ] Images loaded
  - [ ] Status badge shows "Active"

### 4. Verify Data in Firestore
Open Firestore Console → listings collection:
```
Document ID: [listing_id]
Fields:
  - title: [your_title]
  - price: [your_price]
  - ownerId: [your_uid] ← CRITICAL: Must match logged-in user's uid
  - status: "active"
  - images: [array of Firebase Storage URLs]
  - createdAt: [timestamp]
```

### 5. Test Other Listings Show/Hide
- [ ] Create listing as User A
- [ ] Login as User B → doesn't see User A's listing
- [ ] Logout and login as User A → sees only their listing

## Expected Behavior Timeline

```
T+0ms    User clicks "Publish"
T+1s     Form validates & Firestore write starts
T+2s     ✅ Firestore write completes
         ✅ Alert "Success" shown
         
T+3s     User taps "OK"
         ✅ Navigation to /(tabs)/my-listings
         
T+4s     My Listings screen loads
         ✅ Loading spinner appears
         ✅ refreshListings() called
         
T+5s     Firestore query completes
         ✅ Listings loaded from Firestore
         ✅ Filtered by ownerId
         
T+5.5s   Timeout clears loading state
         ✅ Spinner disappears
         ✅ Listing appears in FlatList
```

## If Something Goes Wrong

### "No listings yet" still appears
- Check console logs for ownerId mismatch
- Verify Firestore has `ownerId` field in listing document
- Confirm `currentUser.uid` matches Firestore `ownerId`
- Check `ListingContext` is filtering correctly

### Listing doesn't appear
- Verify Firestore write completed (check console logs)
- Check if listing has `status: "inactive"`
- Verify `ownerId` field exists in Firestore document
- Try logging out and back in to clear cache

### Wrong route error
- Check that `(tabs)/my-listings.tsx` exists
- Verify it exports `MyListingsScreen` from `(owner)/my-listings.tsx`
- Clear app cache: `expo start -c`

### Images not loading
- Check Firestore `images` field has Firebase Storage URLs
- Verify images array is not empty
- Check Firebase Storage rules allow read access

## Files Changed

- [app/(owner)/add-listing.tsx](app/(owner)/add-listing.tsx#L634-L645) - Navigation fix
- [app/my-listings.tsx](app/my-listings.tsx) - Redirect component
- [app/(owner)/my-listings.tsx](app/(owner)/my-listings.tsx#L28-L35) - Loading state

## Rollback (if needed)

Git commands to revert:
```bash
git diff  # See changes
git checkout -- app/(owner)/add-listing.tsx  # Revert one file
git checkout -- app/my-listings.tsx
git checkout -- app/(owner)/my-listings.tsx
```
