# ✅ Selfie Feature - Checklist d'activation

## 📋 Fichiers créés/modifiés

### ✨ Nouveaux fichiers
- ✅ `components/SelfieCapture.tsx` - Composant principal de capture (mobile)
- ✅ `components/SelfieCamera.web.tsx` - Alternative pour web (optionnel)
- ✅ `SELFIE_SETUP.md` - Guide complet de configuration
- ✅ `selfie-migration.sql` - Script de migration DB

### 🔧 Fichiers modifiés
- ✅ `app/profile/index.tsx` - Intégration du composant + UI
- ✅ `lib/database.types.tsx` - Ajout du type `selfie_url`
- ✅ `types/index.tsx` - Ajout de `selfie_url?` au User interface
- ✅ `utils/selfieUpload.tsx` - Déjà existant ✓ (aucune modification nécessaire)

## 🚀 Étapes d'activation

### 1. Database Setup
```sql
-- Exécuter dans Supabase SQL Editor
ALTER TABLE profiles 
ADD COLUMN selfie_url TEXT DEFAULT NULL;
```

### 2. Supabase Storage Configuration
- [ ] Créer le bucket `selfies` dans Storage
  - Public: OFF ❌
- [ ] Configurer les RLS Policies (voir SELFIE_SETUP.md)

### 3. Permissions (Android + iOS)
- [ ] Vérifier `app.json` pour les permissions caméra
- [ ] Vérifier `app.json` pour les permissions galerie

### 4. Test local
```bash
# Web
npm run start-web

# Mobile (iOS)
npm start -- --ios

# Mobile (Android)
npm start -- --android
```

### 5. Vérification du flux
- [ ] Ouvrir la page Profile
- [ ] Cliquer sur "Ajouter un selfie"
- [ ] Prendre une photo ou sélectionner une image
- [ ] Confirmer l'upload
- [ ] Vérifier que `selfie_url` est sauvegardé dans la DB
- [ ] Vérifier que la photo s'affiche sur la page profil

## 🔒 Sécurité

### RLS Policies (Storage)
Les trois policies suivantes **DOIVENT** être configurées:

```
1. INSERT: Users can upload to their own folder
   Formula: (bucket_id = 'selfies') AND (auth.uid()::text = (storage.foldername(name))[1])

2. SELECT: Users can view their own selfies
   Formula: (bucket_id = 'selfies') AND (auth.uid()::text = (storage.foldername(name))[1])

3. DELETE: Users can delete their own selfies
   Formula: (bucket_id = 'selfies') AND (auth.uid()::text = (storage.foldername(name))[1])
```

## 📱 Plateforme Support

| Plateforme | Support | Notes |
|-----------|---------|-------|
| iOS       | ✅ Full | `expo-image-picker` |
| Android   | ✅ Full | `expo-image-picker` |
| Web       | ✅ Partiel | `expo-image-picker` (file picker) |
| Web (WebRTC) | 🔄 Optional | `SelfieCamera.web.tsx` pour meilleure UX |

## 🔧 Configuration optionnelle

### Compression d'image (recommandé)
```bash
bun add expo-image-manipulator
```

Puis modifier `utils/selfieUpload.tsx` pour compresser avant upload.

### Détection de visage (futur)
Pour vérifier la qualité du selfie avant upload:
```bash
bun add @react-native-ml-kit/face-detection
```

## 🐛 Troubleshooting

### ❌ "Selfie bucket doesn't exist"
1. Allez dans Supabase → Storage
2. Créez un nouveau bucket nommé `selfies`
3. Vérifiez que c'est pas publique

### ❌ "Permission denied" sur upload
1. Vérifiez les RLS policies du bucket
2. Vérifiez que l'utilisateur est loggé
3. Vérifiez que l'utilisateur ID correspond au dossier

### ❌ Caméra ne s'ouvre pas sur mobile
1. Vérifiez les permissions dans `app.json`
2. Acceptez les permissions quand demandé
3. Vérifiez sur un device réel (pas simulateur)

### ❌ Web ne fonctionne pas
1. Utilisez HTTPS (pas localhost HTTP)
2. Vérifiez les permissions du navigateur
3. Testez avec Chrome (Firefox aussi)

## 📊 Flux utilisateur

```
Profile Page
    ↓
  [Ajouter un selfie]
    ↓
  Modal Selfie
    ├─→ [Caméra] → Photo prise
    └─→ [Galerie] → Image sélectionnée
    ↓
  Preview + Compression
    ↓
  [Confirmer] → Upload vers Storage
    ↓
  DB Update (profiles.selfie_url)
    ↓
  Affichage sur Profile
```

## 📈 Prochaines étapes

- [ ] Intégrer avec système KYC (vérification manuelle)
- [ ] Ajouter détection de visage (ML Kit)
- [ ] Ajouter compression d'image auto
- [ ] Ajouter possibilité de supprimer selfie
- [ ] Analytics: tracker upload success rate
- [ ] Cache des images localement

## 💾 Migration de DB existante

Si vous avez déjà des données:

```sql
-- Safe migration (sans impacter les données existantes)
ALTER TABLE profiles 
ADD COLUMN IF NOT EXISTS selfie_url TEXT DEFAULT NULL;
```

## 🎯 Validation finale

```bash
# Vérifier les types TypeScript
npx tsc --noEmit

# Vérifier les erreurs lint
npm run lint

# Test build
npm run build

# Test web
npm run start-web

# Test mobile (iOS)
npm start -- --ios

# Test mobile (Android)
npm start -- --android
```

---

**Status**: ✅ Prêt pour activation

**Dernière mise à jour**: 2024-12-11
