# 🚀 Configuration Mapbox + Push Notifications

## ✅ Ce qui a été configuré

1. ✅ **Packages installés**
   - `expo-notifications` 
   - `expo-device`
   - `react-native-maps` (déjà installé)

2. ✅ **Fichiers créés**
   - `env.example` - Template pour les variables d'environnement
   - `.gitignore` - Protection des secrets
   - `hooks/usePushNotifications.ts` - Hook pour les notifications push
   - `utils/geocoding.ts` - Utilitaires pour géocodage (adresse ↔ coordonnées)
   - `utils/directions.ts` - Calcul d'itinéraires et prix
   - `utils/pushTokens.ts` - Gestion des tokens push dans Supabase

3. ✅ **Logo généré**
   - Logo SafeRide avec le "S" vert moderne
   - Icônes générées pour toutes les plateformes

---

## 📋 ÉTAPES À SUIVRE

### 1️⃣ Créer un fichier `.env` (IMPORTANT)

Créez un fichier `.env` à la racine du projet avec ces variables :

```bash
# Supabase (vous les avez déjà)
EXPO_PUBLIC_SUPABASE_URL=https://xxxxx.supabase.co
EXPO_PUBLIC_SUPABASE_ANON_KEY=eyJhbGci...

# Mapbox (À CRÉER)
EXPO_PUBLIC_MAPBOX_TOKEN=pk.eyJ1...

# Expo
EXPO_PUBLIC_EAS_PROJECT_ID=5d91tlhyebcxherprfw5n
```

---

### 2️⃣ Créer un compte Mapbox et obtenir le token

#### Étape 2.1 : Inscription
1. Allez sur [mapbox.com](https://www.mapbox.com)
2. Cliquez sur **Sign up** (gratuit)
3. Vérifiez votre email

#### Étape 2.2 : Récupérer votre Access Token
1. Allez sur [account.mapbox.com](https://account.mapbox.com)
2. Section **Access tokens**
3. Copiez votre **Default public token**
   - Format: `pk.eyJ1IjoieW91cnVzZXJuYW1lIiwiYSI6ImNsxxxxxxxxxxxxxxxx`

#### Étape 2.3 : Ajouter le token dans `.env`
```bash
EXPO_PUBLIC_MAPBOX_TOKEN=pk.eyJ1... # Votre token ici
```

---

### 3️⃣ Configurer Supabase pour les Push Notifications

#### Créer la table `push_tokens`

Allez dans Supabase SQL Editor et exécutez :

```sql
-- Table pour stocker les tokens push
CREATE TABLE push_tokens (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES profiles(id) ON DELETE CASCADE NOT NULL,
  token TEXT UNIQUE NOT NULL,
  device_type TEXT CHECK (device_type IN ('ios', 'android', 'web')),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  last_used TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_push_tokens_user ON push_tokens(user_id);

-- RLS
ALTER TABLE push_tokens ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Utilisateur gère ses tokens" ON push_tokens
  FOR ALL USING (auth.uid() = user_id);
```

#### Créer la fonction RPC pour récupérer les tokens

```sql
-- Fonction pour récupérer les tokens d'un utilisateur
CREATE OR REPLACE FUNCTION get_user_push_tokens(target_user_id UUID)
RETURNS TABLE (token TEXT) AS $$
BEGIN
  RETURN QUERY
  SELECT pt.token
  FROM push_tokens pt
  WHERE pt.user_id = target_user_id
    AND pt.last_used > NOW() - INTERVAL '30 days';
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

---

### 4️⃣ Créer l'Edge Function pour envoyer les notifications

Dans Supabase, créez une Edge Function `push-notify` :

```typescript
// supabase/functions/push-notify/index.ts
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

Deno.serve(async (req) => {
  const supabaseClient = createClient(
    Deno.env.get('SUPABASE_URL') ?? '',
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? '',
  )

  const { user_id, title, body, data } = await req.json()

  // Récupérer les tokens de l'utilisateur
  const { data: tokens } = await supabaseClient
    .rpc('get_user_push_tokens', { target_user_id: user_id })

  if (!tokens || tokens.length === 0) {
    return new Response(
      JSON.stringify({ error: 'Aucun token trouvé' }),
      { status: 404 }
    )
  }

  // Préparer les messages
  const messages = tokens.map(t => ({
    to: t.token,
    sound: 'default',
    title: title,
    body: body,
    data: data || {},
    priority: 'high',
  }))

  // Envoyer via Expo Push API
  const response = await fetch('https://exp.host/--/api/v2/push/send', {
    method: 'POST',
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(messages)
  })

  const result = await response.json()

  return new Response(
    JSON.stringify({ success: true, sent: messages.length, result }),
    { headers: { 'Content-Type': 'application/json' } }
  )
})
```

---

### 5️⃣ Utiliser les notifications dans votre app

#### Dans votre AppContext ou AuthContext

```typescript
import { usePushNotifications } from '@/hooks/usePushNotifications';
import { savePushToken } from '@/utils/pushTokens';

export const AppProvider = ({ children }) => {
  const { expoPushToken } = usePushNotifications();
  const { user } = useAuth();

  // Sauvegarder le token quand l'utilisateur se connecte
  useEffect(() => {
    if (expoPushToken && user?.id) {
      savePushToken(user.id, expoPushToken);
    }
  }, [expoPushToken, user]);

  return <>{children}</>;
};
```

---

### 6️⃣ Utiliser Mapbox Maps dans votre app

#### Exemple d'utilisation dans `app/home/search.tsx`

```typescript
import MapView, { Marker, PROVIDER_DEFAULT } from 'react-native-maps';
import * as Location from 'expo-location';
import { geocodeAddress } from '@/utils/geocoding';

export default function SearchScreen() {
  const [region, setRegion] = useState({
    latitude: 6.1319,  // Lomé
    longitude: 1.2227,
    latitudeDelta: 0.1,
    longitudeDelta: 0.1,
  });

  // Demander permission localisation
  useEffect(() => {
    (async () => {
      const { status } = await Location.requestForegroundPermissionsAsync();
      if (status === 'granted') {
        const location = await Location.getCurrentPositionAsync({});
        setRegion({
          latitude: location.coords.latitude,
          longitude: location.coords.longitude,
          latitudeDelta: 0.05,
          longitudeDelta: 0.05,
        });
      }
    })();
  }, []);

  return (
    <MapView
      provider={PROVIDER_DEFAULT}
      style={{ flex: 1 }}
      region={region}
      showsUserLocation
      showsMyLocationButton
    >
      <Marker
        coordinate={{ latitude: 6.1319, longitude: 1.2227 }}
        title="Lomé Centre"
      />
    </MapView>
  );
}
```

---

## 🧪 Tester les notifications

### Test local

```typescript
import * as Notifications from 'expo-notifications';

// Tester une notification locale
await Notifications.scheduleNotificationAsync({
  content: {
    title: "Test 📬",
    body: 'Ceci est un test de notification',
    data: { test: true },
  },
  trigger: { seconds: 2 },
});
```

### Test avec Expo Push Tool

1. Récupérez votre Expo Push Token (il s'affiche dans les logs)
2. Allez sur [expo.dev/notifications](https://expo.dev/notifications)
3. Entrez votre token et envoyez une notification test

---

## 📊 Limites gratuites

| Service | Limite gratuite |
|---------|-----------------|
| **Mapbox Maps** | 50,000 chargements/mois |
| **Mapbox Geocoding** | 100,000 requêtes/mois |
| **Mapbox Directions** | 100,000 requêtes/mois |
| **Expo Push** | Illimité et gratuit |
| **Supabase** | 2M invocations Edge Functions |

---

## ⚠️ Notes importantes

1. **App.json ne peut pas être modifié** - Le plugin `react-native-maps` est déjà configuré
2. **Le selfie fonctionne** - Il utilise la galerie sur web et la caméra sur mobile
3. **Variables d'environnement** - N'oubliez pas de créer le fichier `.env` avec vos tokens
4. **Dev build** - Si vous voulez utiliser react-native-maps natif (avec Mapbox), vous devrez créer un custom dev build

---

## 📱 Prochaines étapes

Une fois que vous avez :
1. ✅ Créé votre compte Mapbox
2. ✅ Ajouté le token dans `.env`
3. ✅ Créé la table `push_tokens` dans Supabase
4. ✅ Créé l'Edge Function `push-notify`

Vous pourrez :
- 🗺️ Afficher des cartes avec position utilisateur
- 📍 Chercher des adresses au Togo
- 🛣️ Calculer des itinéraires et prix
- 📲 Envoyer des notifications push aux utilisateurs
- 🚨 Gérer les alertes SOS en temps réel

---

## ❓ Questions fréquentes

**Q: Le selfie ne marche pas ?**
R: Sur web, utilisez le bouton "Choisir depuis la galerie". Sur mobile, autorisez l'accès à la caméra.

**Q: Les notifications ne marchent pas ?**
R: Vérifiez que vous avez un appareil physique (pas l'émulateur) et que vous avez autorisé les notifications.

**Q: Mapbox n'affiche rien ?**
R: Vérifiez que votre `EXPO_PUBLIC_MAPBOX_TOKEN` est bien défini dans `.env` et qu'il commence par `pk.`

**Q: Comment tester sur mon téléphone ?**
R: Scannez le QR code qui s'affiche quand vous lancez `npm start`

---

Dites-moi quand vous aurez créé votre compte Mapbox pour que je vous aide à continuer ! 🚀
