# Modèles Réutilisables pour Autres Écrans

Copier/adapter ces patterns pour **tous les autres écrans** (signup, profile, contacts, etc).

---

## 1️⃣ Pattern: Formulaire avec Validation

```typescript
import { validation } from '@/utils/validation';
import { errorLogger } from '@/utils/errorLogger';

export default function FormScreen() {
  const [formData, setFormData] = useState({ name: '', email: '', phone: '' });
  const [error, setError] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(false);

  const handleSubmit = async () => {
    setError(null);

    // Valider
    const { valid, errors } = validation.validateProfileData(formData);
    if (!valid) {
      setError(errors[0]);
      return;
    }

    // Sanitizer
    const sanitized = {
      name: validation.sanitizeName(formData.name),
      email: validation.sanitizeEmail(formData.email),
      phone: formData.phone,
    };

    setIsLoading(true);
    try {
      // Faire la requête avec retry
      const result = await retryManager.executeWithRetry(
        () => apiCall(sanitized),
        'form-submit'
      );

      showSuccess('Sauvegardé!');
    } catch (err) {
      const msg = err instanceof Error ? err.message : 'Erreur';
      setError(msg);
      await errorLogger.log({
        level: 'error',
        message: `Form submission failed: ${msg}`,
        metadata: { formType: 'profile' },
      });
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <SafeAreaView>
      {error && <ErrorBanner text={error} />}
      <TextInput value={formData.name} onChangeText={/* ... */} />
      <Button onPress={handleSubmit} disabled={isLoading}>
        {isLoading ? 'Sauvegarde...' : 'Enregistrer'}
      </Button>
    </SafeAreaView>
  );
}
```

---

## 2️⃣ Pattern: Data Listing avec Cache

```typescript
import { ridesCache, profileCache } from '@/utils/cache';
import { useNetworkStatus } from '@/hooks/UseNetworkStatus';

export default function ListScreen() {
  const [items, setItems] = useState<any[]>([]);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const { isOnline } = useNetworkStatus();

  const loadItems = async () => {
    setIsLoading(true);
    setError(null);

    try {
      // Essayer cache d'abord
      let data = await ridesCache.getRides();

      if (!data && isOnline) {
        // Fetch serveur avec retry
        data = await retryManager.executeWithRetry(
          () => supabase.from('table').select('*'),
          'fetch-items'
        );
        // Cacher pour après
        await ridesCache.setRides(data);
      }

      setItems(data || []);
      if (!data) {
        setError('Aucune donnée disponible');
      }
    } catch (err) {
      await errorLogger.logNetworkError(err instanceof Error ? err.message : 'Unknown');
      setError('Erreur lors du chargement. Utilisation du cache.');
    } finally {
      setIsLoading(false);
    }
  };

  useEffect(() => {
    loadItems();
  }, [isOnline]);

  return (
    <SafeAreaView>
      {!isOnline && <OfflineBanner />}
      {error && <ErrorBanner text={error} />}
      <FlatList
        data={items}
        renderItem={({ item }) => <ListItem item={item} />}
        onRefresh={loadItems}
        refreshing={isLoading}
      />
    </SafeAreaView>
  );
}
```

---

## 3️⃣ Pattern: Real-Time Data Sync

```typescript
import { realtimeManager } from '@/utils/realtimeManager';

export default function RealtimeScreen({ entityId }: { entityId: string }) {
  const [data, setData] = useState<any>(null);

  useEffect(() => {
    // Subscribe à real-time updates
    realtimeManager.subscribe(`entity-${entityId}`, {
      table: 'table_name',
      filter: `id=eq.${entityId}`,
      onInsert: (newData) => {
        setData(newData);
      },
      onUpdate: (updatedData) => {
        setData(updatedData);
      },
      onDelete: (deletedData) => {
        handleDelete(deletedData);
      },
    });

    return () => {
      realtimeManager.unsubscribe(`entity-${entityId}`);
    };
  }, [entityId]);

  return (
    <SafeAreaView>
      {data && <View>{/* afficher data en direct */}</View>}
    </SafeAreaView>
  );
}
```

---

## 4️⃣ Pattern: Offline-First Mutation

```typescript
import { syncQueueService } from '@/utils/syncQueue';
import { useNetworkStatus } from '@/hooks/UseNetworkStatus';

export default function MutationScreen() {
  const { isOnline } = useNetworkStatus();

  const handleUpdate = async (updateData: any) => {
    // Si offline, ajouter à queue (sera syncée auto quand online)
    if (!isOnline) {
      await syncQueueService.addToQueue({
        operation: 'other',
        table: 'table_name',
        action: 'update',
        data: updateData,
        priority: 2,
      });

      showSuccess('Changement sauvegardé. Sera envoyé quand online.');
      return;
    }

    // Si online, faire immédiatement
    try {
      await retryManager.executeWithRetry(
        () => supabase.from('table_name').update(updateData).eq('id', updateData.id),
        'update-mutation'
      );
      showSuccess('Mis à jour!');
    } catch (error) {
      // Fallback: ajouter à queue pour retry plus tard
      await syncQueueService.addToQueue({
        operation: 'other',
        table: 'table_name',
        action: 'update',
        data: updateData,
        priority: 2,
      });
      showError('Erreur. Retry en arrière-plan...');
    }
  };

  return <View>{/* ... */}</View>;
}
```

---

## 5️⃣ Pattern: Rate-Limited Action (SOS-like)

```typescript
import { rateLimiter } from '@/utils/rateLimiter';

export default function RateLimitedScreen({ userId }: { userId: string }) {
  const [isBlocked, setIsBlocked] = useState(false);
  const [remaining, setRemaining] = useState(0);

  const handleAction = async () => {
    // Vérifier si allowed
    const allowed = await rateLimiter.isAllowed(userId, {
      maxRequests: 5,
      windowMs: 60000, // 5 max par minute
      keyPrefix: 'custom_action',
    });

    if (!allowed) {
      const rem = await rateLimiter.getRemainingRequests(userId);
      setRemaining(rem);
      setIsBlocked(true);
      return;
    }

    // Faire action
    try {
      await myAction();
      showSuccess('Action effectuée!');
    } catch (error) {
      await errorLogger.log({
        level: 'error',
        message: `Action failed`,
      });
    }
  };

  return (
    <SafeAreaView>
      {isBlocked && <Text>Trop d'actions. Attendez {remaining}s</Text>}
      <Button onPress={handleAction} disabled={isBlocked}>
        Effectuer action
      </Button>
    </SafeAreaView>
  );
}
```

---

## 6️⃣ Pattern: Error Boundary & Logging

```typescript
import { errorLogger } from '@/utils/errorLogger';

export default function SafeScreen() {
  const [error, setError] = useState<string | null>(null);

  const safeAsyncOperation = async () => {
    try {
      // Faire quelque chose
      const result = await dangerousOperation();
      return result;
    } catch (err) {
      // Logger l'erreur
      const error = err instanceof Error ? err : new Error(String(err));

      await errorLogger.log({
        level: 'error',
        message: `Operation failed: ${error.message}`,
        stack: error.stack,
        userId: userId,
        rideId: rideId,
        metadata: {
          operation: 'critical_task',
          timestamp: new Date().toISOString(),
        },
      });

      setError(error.message);
      throw error;
    }
  };

  return (
    <SafeAreaView>
      {error && <ErrorBanner text={error} onDismiss={() => setError(null)} />}
    </SafeAreaView>
  );
}
```

---

## 🎯 Résumé: Imports Standards

Tous les écrans doivent importer:

```typescript
// Validation & sanitization
import { validation } from '@/utils/validation';

// Error logging
import { errorLogger } from '@/utils/errorLogger';

// Retry logic
import { retryManager } from '@/utils/retryManager';

// Network awareness
import { useNetworkStatus } from '@/hooks/UseNetworkStatus';

// Optionnel selon besoin:
import { ridesCache, profileCache } from '@/utils/cache';
import { realtimeManager } from '@/utils/realtimeManager';
import { syncQueueService } from '@/utils/syncQueue';
import { rateLimiter } from '@/utils/rateLimiter';
```

---

## 📋 Checklist pour Chaque Écran

- [ ] Validation input (si formulaire)
- [ ] Error banner affichée
- [ ] Loading state géré
- [ ] Retry logic sur appels API
- [ ] Error logged si fail
- [ ] Offline mode handled (cache ou queue)
- [ ] Real-time updates (si applicable)
- [ ] Rate limiting (si action sensible)
- [ ] Network status affiché
- [ ] Lint passe

---

## 🔄 Appliquer à Tous les Écrans Manquants

**Écrans à traiter:**
- `app/auth/signup.tsx` → ajouter validation
- `app/auth/personal-info.tsx` → validation + cache profile
- `app/auth/verify-email.tsx` → retry logic
- `app/home/search.tsx` → cache + realtime
- `app/home/driver-details.tsx` → realtime profile updates
- `app/profile/index.tsx` → validation + offline mutations
- `app/profile/emergency-contacts.tsx` → validation + cache
- `app/admin/*` → error logging + realtime

**Temps estimé:** 30min par écran avec ce template.

---

## 💡 Pro Tips

1. **Ctrl+Shift+F** pour remplacer patterns sur tous les écrans
2. Utiliser ce fichier comme copier/coller template
3. Toujours tester offline mode avant de merger
4. Sentry dashboard pour monitorer erreurs en production
5. `INTEGRATION_GUIDE.md` pour exemples avancés

