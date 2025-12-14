# Robustness Features Integration Guide

Quick examples for integrating robustness features into your screens.

## 1. Booking Screen Integration

```typescript
import { useSafeBooking } from '@/hooks/UseSafeBooking';
import { validation } from '@/utils/validation';
import { errorLogger } from '@/utils/errorLogger';

export function BookingScreen({ rideId }: { rideId: string }) {
  const { createBooking, isLoading, error } = useSafeBooking();
  const { profile } = useAuth();

  const handleBook = async () => {
    // Validate inputs
    if (!validation.isValidUUID(rideId) || !validation.isValidUUID(profile?.id)) {
      showError('Invalid ride or user ID');
      return;
    }

    // Create booking (uses Edge Function + rate limiting)
    const result = await createBooking(rideId, profile!.id);

    if (!result.success) {
      showError(result.error);
      return;
    }

    showSuccess('Booking confirmed!');
    navigate('RideDetails', { rideId });
  };

  return (
    <Button 
      disabled={isLoading} 
      onPress={handleBook}
    >
      {isLoading ? 'Booking...' : 'Confirm Booking'}
    </Button>
  );
}
```

## 2. Active Ride Tracking (Location)

```typescript
import { useLocationTracking } from '@/hooks/UseLocationTracking';
import { useNetworkStatus } from '@/hooks/UseNetworkStatus';

export function ActiveRideScreen({ rideId }: { rideId: string }) {
  const { location, isTracking, error } = useLocationTracking(true, rideId);
  const { isOnline } = useNetworkStatus();

  useEffect(() => {
    if (error) {
      errorLogger.logNetworkError(`Location tracking error: ${error}`);
    }
  }, [error]);

  return (
    <View>
      {!isOnline && <OfflineBanner />}
      {isTracking && <LocationIndicator location={location} />}
      <MapView
        initialRegion={location ? { 
          latitude: location.latitude,
          longitude: location.longitude,
          latitudeDelta: 0.01,
          longitudeDelta: 0.01,
        } : DEFAULT_REGION}
      />
    </View>
  );
}
```

## 3. Real-Time Ride Updates

```typescript
import { realtimeManager } from '@/utils/realtimeManager';
import { useEffect, useState } from 'react';

export function RideDetailsScreen({ rideId }: { rideId: string }) {
  const [ride, setRide] = useState(null);

  useEffect(() => {
    // Subscribe to ride updates
    realtimeManager.subscribe(`ride-${rideId}`, {
      table: 'rides',
      filter: `id=eq.${rideId}`,
      onUpdate: (updatedRide) => {
        setRide(updatedRide);
      },
    });

    return () => {
      realtimeManager.unsubscribe(`ride-${rideId}`);
    };
  }, [rideId]);

  return (
    <View>
      {ride && (
        <>
          <Text>Driver: {ride.driver_name}</Text>
          <Text>Status: {ride.status}</Text>
          <Text>Price: {ride.price} XOF</Text>
        </>
      )}
    </View>
  );
}
```

## 4. Rides List with Caching

```typescript
import { ridesCache } from '@/utils/cache';
import { retryManager } from '@/utils/retryManager';
import { useNetworkStatus } from '@/hooks/UseNetworkStatus';

export function RidesListScreen() {
  const [rides, setRides] = useState([]);
  const [isLoading, setIsLoading] = useState(false);
  const { isOnline } = useNetworkStatus();

  const loadRides = async () => {
    setIsLoading(true);
    try {
      // Try cache first
      let ridesData = await ridesCache.getRides();

      if (!ridesData && isOnline) {
        // Fetch from server with retry
        ridesData = await retryManager.executeWithRetry(
          async () => {
            const { data, error } = await supabase
              .from('rides')
              .select('*')
              .eq('status', 'published');
            if (error) throw error;
            return data;
          },
          'fetch-published-rides'
        );

        // Cache for next time
        await ridesCache.setRides(ridesData);
      }

      setRides(ridesData || []);
    } catch (error) {
      errorLogger.logNetworkError('Failed to load rides');
      showError('Unable to load rides. Using cached data.');
    } finally {
      setIsLoading(false);
    }
  };

  useEffect(() => {
    loadRides();
  }, [isOnline]);

  return (
    <FlatList
      data={rides}
      renderItem={({ item }) => <RideCard ride={item} />}
      onRefresh={loadRides}
      refreshing={isLoading}
    />
  );
}
```

## 5. Profile Update with Validation

```typescript
import { validation } from '@/utils/validation';
import { syncQueueService } from '@/utils/syncQueue';
import { profileCache } from '@/utils/cache';

export function ProfileEditScreen() {
  const { profile, updateProfile } = useAuth();
  const [name, setName] = useState(profile?.name);
  const [email, setEmail] = useState(profile?.email);

  const handleSave = async () => {
    // Validate
    const { valid, errors } = validation.validateProfileData({
      name,
      email,
      role: profile?.role,
    });

    if (!valid) {
      showError(errors[0]);
      return;
    }

    // Sanitize
    const sanitizedName = validation.sanitizeName(name);

    try {
      // Update with retry
      await retryManager.executeWithRetry(
        () => updateProfile({ 
          name: sanitizedName, 
          email: validation.sanitizeEmail(email)
        }),
        'update-profile'
      );

      // Clear cache for fresh data
      await profileCache.clearProfile(profile!.id);

      showSuccess('Profile updated!');
    } catch (error) {
      errorLogger.logError(
        error instanceof Error ? error : new Error('Update failed'),
        { action: 'profile-update' }
      );
    }
  };

  return (
    <ScrollView>
      <TextInput
        value={name}
        onChangeText={setName}
        placeholder="Full Name"
        maxLength={100}
      />
      <TextInput
        value={email}
        onChangeText={setEmail}
        placeholder="Email"
        keyboardType="email-address"
      />
      <Button onPress={handleSave}>Save Profile</Button>
    </ScrollView>
  );
}
```

## 6. SOS Alert with Rate Limiting

```typescript
import { sosRateLimit } from '@/utils/rateLimiter';
import { errorLogger } from '@/utils/errorLogger';

export function SOSModal({ userId, rideId }: Props) {
  const [isLoading, setIsLoading] = useState(false);
  const { location } = useLocationTracking(true);

  const handleSOS = async () => {
    // Check rate limit
    const canAlert = await sosRateLimit.canAlert(userId);
    if (!canAlert) {
      showError('SOS cooldown active. Contact support if urgent.');
      return;
    }

    setIsLoading(true);
    try {
      // Create SOS with retry (high priority)
      const { data, error } = await retryManager.executeWithRetry(
        () => supabase.from('sos_alerts').insert({
          user_id: userId,
          ride_id: rideId,
          latitude: location?.latitude || 0,
          longitude: location?.longitude || 0,
          status: 'active',
        }),
        'create-sos-alert'
      );

      if (error) throw error;

      // Log critical error
      await errorLogger.logSOSError('SOS triggered', data[0].id, userId);

      showSuccess('Alert sent to support. Stay safe.');
      closeSOS();
    } catch (error) {
      errorLogger.logSOSError('SOS failed to send', '', userId);
      showError('Failed to send alert. Calling emergency contact...');
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <Modal>
      <Pressable onPress={handleSOS} disabled={isLoading}>
        <Text>{isLoading ? 'Sending Alert...' : 'SEND SOS'}</Text>
      </Pressable>
    </Modal>
  );
}
```

## 7. Network Error Handling (Generic Pattern)

```typescript
import { useNetworkStatus } from '@/hooks/UseNetworkStatus';
import { retryManager } from '@/utils/retryManager';

export function useAsync<T>(asyncFunction: () => Promise<T>) {
  const [state, setState] = useState<'pending' | 'success' | 'error'>('pending');
  const [data, setData] = useState<T | null>(null);
  const [error, setError] = useState<Error | null>(null);
  const { isOnline } = useNetworkStatus();

  const execute = useCallback(async () => {
    if (!isOnline) {
      setError(new Error('No internet connection'));
      setState('error');
      return;
    }

    setState('pending');
    try {
      const result = await retryManager.executeWithRetry(
        asyncFunction,
        'async-operation'
      );
      setData(result);
      setState('success');
    } catch (err) {
      const error = err instanceof Error ? err : new Error('Unknown error');
      setError(error);
      setState('error');
      errorLogger.logNetworkError(error.message);
    }
  }, [asyncFunction, isOnline]);

  return { execute, state, data, error };
}
```

## 8. Initialization in App Root

```typescript
// app/_layout.tsx
import { initSentry } from '@/utils/sentry';
import { AuthProvider } from '@/contexts/AuthContext';
import { AppProvider } from '@/contexts/AppContext';

export default function RootLayout() {
  useEffect(() => {
    // Initialize error monitoring
    initSentry();
  }, []);

  return (
    <AuthProvider>
      <AppProvider>
        <RootNavigator />
      </AppProvider>
    </AuthProvider>
  );
}
```

## Key Integration Patterns

### Pattern 1: Fetch with Cache & Retry
```typescript
const cached = await someCache.get(key);
if (cached) return cached;

const fresh = await retryManager.executeWithRetry(
  fetchOperation,
  'operation-name'
);

await someCache.set(key, fresh);
return fresh;
```

### Pattern 2: Real-Time Subscription
```typescript
realtimeManager.subscribe('key', {
  table: 'table_name',
  filter: `column=eq.value`,
  onUpdate: (data) => { /* handle */ },
  onInsert: (data) => { /* handle */ },
  onDelete: (data) => { /* handle */ },
});

// Cleanup
return () => realtimeManager.unsubscribe('key');
```

### Pattern 3: Offline-First Mutation
```typescript
// Add to sync queue if offline
await syncQueueService.addToQueue({
  operation: 'booking',
  table: 'bookings',
  action: 'insert',
  data: { /* booking data */ },
  priority: 3,
});

// Sync when online (automatic or manual)
if (isOnline) await syncQueueService.syncQueue();
```

### Pattern 4: Error Context
```typescript
await errorLogger.log({
  level: 'error',
  message: 'Descriptive error message',
  userId: user?.id,
  rideId: ride?.id,
  metadata: { /* additional context */ },
});
```

---

## Testing Integration

```bash
# Run RLS security tests
npm run test:rls

# Run booking E2E tests
npm run test:booking

# Run linter
expo lint

# Test build locally
eas build --platform android --profile staging --local
```
