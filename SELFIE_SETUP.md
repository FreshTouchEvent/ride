# Configuration Selfie (Mobile + Web)

## 1️⃣ Configuration Supabase

### A. Créer le bucket Storage

1. Allez dans **Supabase Dashboard** → **Storage**
2. Cliquez sur **Create Bucket**
3. Configurez:
   - **Bucket Name**: `selfies`
   - **Public bucket**: ❌ (OFF - Pour la sécurité, utiliser des URL publiques via RLS)

### B. Ajouter la colonne à la base de données

Exécutez ce SQL dans **Supabase SQL Editor**:

```sql
ALTER TABLE profiles 
ADD COLUMN selfie_url TEXT DEFAULT NULL;
```

### C. Configurer les politiques de sécurité (RLS)

#### 1. Users can upload their own selfies
- **Bucket**: `selfies`
- **Operation**: INSERT
- **Role**: authenticated
- **Policy Expression**:
```sql
(bucket_id = 'selfies') AND (auth.uid()::text = (storage.foldername(name))[1])
```

#### 2. Users can view their own selfies
- **Bucket**: `selfies`
- **Operation**: SELECT
- **Role**: authenticated
- **Policy Expression**:
```sql
(bucket_id = 'selfies') AND (auth.uid()::text = (storage.foldername(name))[1])
```

#### 3. Users can delete their own selfies
- **Bucket**: `selfies`
- **Operation**: DELETE
- **Role**: authenticated
- **Policy Expression**:
```sql
(bucket_id = 'selfies') AND (auth.uid()::text = (storage.foldername(name))[1])
```

## 2️⃣ Installation des dépendances

Les dépendances requises sont déjà installées:
- ✅ `expo-image-picker` - Capture et sélection d'images
- ✅ `@supabase/supabase-js` - Upload vers Storage

## 3️⃣ Intégration dans l'app

### Structure des fichiers créés:

```
components/
├── SelfieCapture.tsx        # Composant principal (mobile)
└── SelfieCamera.web.tsx     # Alternative web (optionnel)

utils/
└── selfieUpload.tsx         # Utilitaires d'upload (existant)

app/profile/
└── index.tsx                # Page profile intégrée
```

### Utilisation dans un composant:

```tsx
import SelfieCapture from '@/components/SelfieCapture';

export default function MyComponent() {
  const handleSelfieSuccess = (url: string) => {
    console.log('Selfie URL:', url);
  };

  const handleSelfieError = (error: string) => {
    console.log('Error:', error);
  };

  return (
    <SelfieCapture
      userId={user.id}
      onSuccess={handleSelfieSuccess}
      onError={handleSelfieError}
    />
  );
}
```

## 4️⃣ Permissions requises

### Android (`app.json`)
```json
{
  "plugins": [
    [
      "expo-image-picker",
      {
        "photosPermission": "Autorisez l'accès à la galerie photo",
        "cameraPermission": "Autorisez l'accès à la caméra"
      }
    ]
  ]
}
```

### iOS (`app.json`)
```json
{
  "infoPlist": {
    "NSCameraUsageDescription": "Autorisez l'accès à la caméra pour prendre un selfie",
    "NSPhotoLibraryUsageDescription": "Autorisez l'accès à la galerie photo"
  }
}
```

## 5️⃣ Structure des fichiers dans le bucket

```
selfies/
├── {user-id-1}/
│   ├── {user-id-1}-1700000000000.jpg
│   ├── {user-id-1}-1700000001000.jpg
│   └── ...
├── {user-id-2}/
│   ├── {user-id-2}-1700000000000.jpg
│   └── ...
```

**Chaque utilisateur ne peut voir/modifier que ses propres fichiers** grâce aux RLS policies.

## 6️⃣ Vérification KYC (Optionnel)

### Workflow suggéré:
1. **Selfie capturé** → `selfie_url` sauvegardé dans `profiles`
2. **Admin review** → Via Retool ou dashboard custom
3. **KYC Status** → Mis à jour: `pending` → `approved` | `rejected`
4. **User notification** → Via email/SMS

### Table des KYC statuts:
```sql
kyc_status IN ('pending', 'approved', 'rejected')
```

## 7️⃣ Compression d'image (Optionnel)

Pour réduire la taille des uploads, installez:
```bash
bun add expo-image-manipulator
```

Puis dans `utils/selfieUpload.tsx`:
```tsx
import * as ImageManipulator from 'expo-image-manipulator';

const manipulatedImage = await ImageManipulator.manipulateAsync(
  imageUri,
  [{ resize: { width: 1024, height: 1024 } }],
  { compress: 0.7, format: 'jpeg' }
);
```

## 8️⃣ Troubleshooting

### Erreur: "Bucket doesn't exist"
- Vérifiez que le bucket `selfies` est créé dans Storage
- Vérifiez le nom exact du bucket

### Erreur: "Permission denied"
- Vérifiez les RLS policies du bucket
- Vérifiez que l'utilisateur est authentifié

### Erreur: "File too large"
- Comprimez l'image avant upload
- Utilisez `expo-image-manipulator` (voir section 7)

### Caméra ne fonctionne pas sur web
- Utilisez HTTPS (pas HTTP)
- Vérifiez les permissions du navigateur
- Testez avec Chrome/Firefox (Safari a des limitations)

## 9️⃣ Migration de la base de données

Si vous avez une base de données existante:

```sql
ALTER TABLE profiles 
ADD COLUMN IF NOT EXISTS selfie_url TEXT DEFAULT NULL;
```

Aucun impact sur les données existantes.
