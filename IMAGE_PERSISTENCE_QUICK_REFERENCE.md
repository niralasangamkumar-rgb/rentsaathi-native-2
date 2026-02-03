# Image Persistence Bug Fix - Quick Reference

## Problem
Previously added listings lost their images when new listings were published.

## Root Cause
App was saving local file:// URIs directly to Firestore instead of uploading images to Firebase Storage.

## Solution Summary

### 1. Upload Flow
- User selects photos → Immediately upload to Firebase Storage → Get download URLs → Save URLs to Firestore

### 2. Storage Path Structure
```
listings/{listingId}/{timestamp}.jpg
```
Example: `listings/1706700123456/1706700124567.jpg`

### 3. Firestore Data
Only store Firebase Storage download URLs:
```json
{
  "images": [
    "https://firebasestorage.googleapis.com/.../listings/1706700123456/1706700124567.jpg",
    "https://firebasestorage.googleapis.com/.../listings/1706700123456/1706700125789.jpg"
  ]
}
```

### 4. Key Code Changes

**Image Upload Function:**
```typescript
const uploadImageToStorage = async (imageUri: string, listingId: string): Promise<string> => {
  const blob = await (await fetch(imageUri)).blob();
  const path = `listings/${listingId}/${Date.now()}.jpg`;
  const ref = ref(storage, path);
  await uploadBytes(ref, blob, { contentType: 'image/jpeg' });
  return await getDownloadURL(ref);
};
```

**Upload During Image Selection:**
```typescript
const downloadURLs = [];
for (const uri of selectedUris) {
  const url = await uploadImageToStorage(uri, tempListingId);
  downloadURLs.push(url);
}
setPhotos([...photos, ...downloadURLs]);
```

**Save to Firestore - Edit Mode:**
```typescript
const existingImages = listingToEdit.images || [];
const newImages = photos.filter(p => p.startsWith('https://'));
const finalImages = [...existingImages, ...newImages];
await updateDoc(listingRef, { images: finalImages });
```

**Normalize on Read:**
```typescript
const normalizedImages = Array.isArray(data.images) ? data.images : 
                        Array.isArray(data.imageUrls) ? data.imageUrls : [];
```

## Testing

✅ Create new listing with images → Images persist  
✅ Edit listing to add images → Old + new images show  
✅ Force close app → Images still load  
✅ Multiple listings → Each shows correct images  

## Console Logs to Watch For

```
[ImageUpload] Starting upload for listing {id}
[ImageUpload] Upload successful. Download URL: https://firebasestorage.googleapis.com/...
[AddListing] Publishing with {count} images
[AddListing] CONFIRMED: Images saved to Firestore: {count}
[ListingContext] Listing {id}: Found {count} images in 'images' field
```

## Firebase Security

Images are public (anyone can read), but only authenticated users can upload.

## Migration

Old listings with local file:// URIs will:
- Continue to work until app reinstall
- Get proper Firebase Storage URLs when owner edits
- Display correctly with fallback to placeholder if URL breaks

## Key Files

- `config/firebase.ts` - Firebase Storage initialization
- `app/(owner)/add-listing.tsx` - Upload logic + form handling
- `context/ListingContext.tsx` - Data normalization
- `app/listing/[id].tsx` - Detail view rendering
- `app/(owner)/my-listings.tsx` - Owner's listings
- `app/(tabs)/home.tsx` - Feed/home listings
