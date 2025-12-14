# 🧪 Guide de test - Selfie Feature

## Configuration initiale

### 1. Database
```sql
-- Exécuter dans Supabase SQL Editor
ALTER TABLE profiles 
ADD COLUMN IF NOT EXISTS selfie_url TEXT DEFAULT NULL;
```

### 2. Supabase Storage
- Créer bucket: `selfies`
- Appliquer les 3 RLS policies (voir SELFIE_SETUP.md)

### 3. Installation locale
```bash
cd React_TGRide
bun install
```

## Test Web

### Démarrage
```bash
npm run start-web
```

### Étapes de test
1. Allez à **Profile** (menu ou route `/profile`)
2. Cherchez la section **"Ajouter un selfie"**
3. Cliquez sur le bouton
4. Sélectionnez ou capturez une image
5. Confirmez l'upload
6. Vérifiez que:
   - ✅ La photo s'affiche sur la carte profil
   - ✅ L'URL est sauvegardée dans DB (profiles.selfie_url)
   - ✅ Aucune erreur dans la console

### Cas de test

#### ✅ Cas nominal
- [ ] Taille image < 5MB
- [ ] Format: JPG/PNG
- [ ] Upload réussit avec message de succès
- [ ] Photo affichée immédiatement

#### ❌ Cas erreur
- [ ] Image > 5MB → Erreur affichée
- [ ] Pas de permissions → Message d'erreur
- [ ] Utilisateur non loggé → Redirection login
- [ ] Connexion Internet perdue → Erreur d'upload

#### 🔒 Sécurité
- [ ] Autre user ne peut pas voir les selfies d'un utilisateur
- [ ] Autre user ne peut pas supprimer un selfie
- [ ] URL fichier suit pattern: `selfies/{user_id}/{user_id}-{timestamp}.jpg`

## Test Mobile iOS

### Démarrage
```bash
npm start -- --ios
```

### Étapes de test
1. L'app s'ouvre dans le simulateur
2. Allez à **Profile**
3. Cliquez sur **"Ajouter un selfie"**
4. Testez les deux options:
   - [ ] **Caméra**: Prend une photo directement
   - [ ] **Galerie**: Sélectionne une image existante
5. Confirmez le selfie
6. Vérifiez:
   - ✅ Photo s'affiche correctement
   - ✅ Aucun crash sur permissions
   - ✅ Compression fonctionne

### Permissions
- La première fois, l'app demande les permissions
- Acceptez:
  - [ ] Camera access
  - [ ] Photo library access

## Test Mobile Android

### Démarrage
```bash
npm start -- --android
```

### Étapes similaires à iOS
1. Application dans l'émulateur
2. Profile → Ajouter un selfie
3. Testez **Caméra** et **Galerie**
4. Confirmez l'upload

### Spécificités Android
- Les permissions demandées lors du runtime
- Vérifiez dans Settings → Permissions si refusées

## Cas d'utilisation avancés

### Remplacement de selfie
- [ ] Upload un premier selfie
- [ ] Upload un deuxième → L'ancien est remplacé
- [ ] Vérifiez: URL mise à jour dans DB, ancienne photo supprimée

### Suppression (futur)
```tsx
// Quand implémenté
<TouchableOpacity onPress={() => deleteSelfie(userId)}>
  <Text>Supprimer le selfie</Text>
</TouchableOpacity>
```

### Affichage optimisé
- [ ] Image se charge rapidement
- [ ] Pas de freeze UI lors du chargement
- [ ] Cache fonctionne sur rechargement

## Vérifications de sécurité

### RLS Policies
```sql
-- Vérifier que ces policies existent:
SELECT * FROM pg_policies WHERE tablename = 'objects' AND schemaname = 'storage';
```

### Test de sécurité
1. Loggez-vous avec User A
2. Uploadez un selfie
3. Ouvrez DevTools → Network
4. Cherchez l'URL du selfie
5. Loggez-vous avec User B
6. Essayez d'accéder à l'URL du selfie de User A
   - Doit être **BLOQUEÉ** par RLS ❌

## Performance

### Métriques à vérifier
- [ ] Upload < 3s pour une image 1MB
- [ ] Memory usage stable pendant capture
- [ ] Pas de memory leak après plusieurs uploads

### Tools
```bash
# React Profiler
React DevTools → Profiler → Record selfie upload

# Network
DevTools → Network → Monitor upload progress
```

## Logs et debugging

### Vérifier les logs
```javascript
// Dans le composant
console.log('📸 Début de l\'upload du selfie...');
console.log('👤 User ID:', userId);
console.log('📁 Image URI:', imageUri);
```

### Supabase logs
1. Allez dans Supabase Dashboard → Logs
2. Cherchez les erreurs d'upload
3. Vérifiez les RLS violations

## Checklist finale

- [ ] Web: Upload + affichage ✅
- [ ] iOS: Camera + Gallery ✅
- [ ] Android: Camera + Gallery ✅
- [ ] DB: URL sauvegardée ✅
- [ ] Security: RLS policies appliquées ✅
- [ ] Error handling: Messages d'erreur affichés ✅
- [ ] Permissions: Demandées correctement ✅
- [ ] Performance: Pas de lag ou crash ✅

## Rapport de test

Créez un fichier `SELFIE_TEST_REPORT.md`:

```markdown
# Test Report - Selfie Feature

## Date
- 2024-12-11

## Tester
- [Votre nom]

## Platform
- [ ] Web (Chrome/Firefox)
- [ ] iOS (Real device / Simulator)
- [ ] Android (Real device / Emulator)

## Results
- Upload: PASS/FAIL
- Display: PASS/FAIL
- Security: PASS/FAIL
- Performance: PASS/FAIL

## Issues found
- [ ] None
- [ ] Minor: ...
- [ ] Critical: ...

## Sign-off
Tester: _______________
Date: _______________
```

---

**Tip**: Lors du test, gardez les DevTools ouverts pour voir les logs et les erreurs réseau.
