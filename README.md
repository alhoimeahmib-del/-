# -
مودات مامودات ما// App.js
import React, { useEffect, useState } from "react";
import { View, Text, FlatList, TouchableOpacity, Image, Alert, StyleSheet } from "react-native";
import * as ImagePicker from "expo-image-picker";
import * as FileSystem from "expo-file-system";
import AsyncStorage from "@react-native-async-storage/async-storage";

const NUM_MODS = 50;
const STORAGE_KEY = "mods_data_v1";

// create 50 placeholder mods
const createDefaultMods = () => {
  const mods = [];
  for (let i = 1; i <= NUM_MODS; i++) {
    mods.push({
      id: `mod-${i}`,
      name: `مود ${i}`,
      shortDesc: `وصف مختصر للمود ${i}`,
      imageUri: null, // user will assign an image
    });
  }
  return mods;
};

export default function App() {
  const [mods, setMods] = useState([]);

  useEffect(() => {
    (async () => {
      const saved = await AsyncStorage.getItem(STORAGE_KEY);
      if (saved) {
        try {
          setMods(JSON.parse(saved));
          return;
        } catch {}
      }
      const defaults = createDefaultMods();
      setMods(defaults);
      await AsyncStorage.setItem(STORAGE_KEY, JSON.stringify(defaults));
    })();
  }, []);

  // request permissions for image picker
  useEffect(() => {
    (async () => {
      const { status } = await ImagePicker.requestMediaLibraryPermissionsAsync();
      if (status !== "granted") {
        Alert.alert("صلاحيات مطلوبة", "التطبيق يحتاج صلاحية الوصول إلى الصور لاستيراد صور المودات.");
      }
    })();
  }, []);

  const pickImageForMod = async (modId) => {
    try {
      const result = await ImagePicker.launchImageLibraryAsync({
        mediaTypes: ImagePicker.MediaTypeOptions.Images,
        quality: 0.9,
      });

      if (result.cancelled) return;

      // copy selected file into app's document directory to keep persistent copy
      const sourceUri = result.uri;
      const destDir = `${FileSystem.documentDirectory}mod_images/`;
      await FileSystem.makeDirectoryAsync(destDir, { intermediates: true });
      const filename = `${modId}_${Date.now()}.jpg`;
      const destUri = destDir + filename;
      await FileSystem.copyAsync({ from: sourceUri, to: destUri });

      const updated = mods.map((m) => (m.id === modId ? { ...m, imageUri: destUri } : m));
      setMods(updated);
      await AsyncStorage.setItem(STORAGE_KEY, JSON.stringify(updated));
    } catch (e) {
      Alert.alert("خطأ", "فشل استيراد الصورة.");
      console.error(e);
    }
  };

  const clearImage = async (modId) => {
    const updated = mods.map((m) => (m.id === modId ? { ...m, imageUri: null } : m));
    setMods(updated);
    await AsyncStorage.setItem(STORAGE_KEY, JSON.stringify(updated));
  };

  const exportModsJson = async () => {
    // creates a JSON file listing mods and image paths (useful للنسخ الاحتياطي)
    const json = JSON.stringify(mods, null, 2);
    const path = `${FileSystem.documentDirectory}mods_export_${Date.now()}.json`;
    await FileSystem.writeAsStringAsync(path, json, { encoding: FileSystem.EncodingType.UTF8 });
    Alert.alert("تم التصدير", `ملف JSON محفوظ هنا:\n${path}`);
  };

  const renderItem = ({ item }) => (
    <View style={styles.card}>
      <View style={{ flex: 1 }}>
        <Text style={styles.title}>{item.name}</Text>
        <Text style={styles.desc}>{item.shortDesc}</Text>
        <View style={{ flexDirection: "row", marginTop: 8 }}>
          <TouchableOpacity style={styles.button} onPress={() => pickImageForMod(item.id)}>
            <Text style={styles.btnText}>استيراد صورة</Text>
          </TouchableOpacity>
          <TouchableOpacity style={[styles.button, { backgroundColor: "#bbb", marginLeft: 8 }]} onPress={() => clearImage(item.id)}>
            <Text style={styles.btnText}>مسح الصورة</Text>
          </TouchableOpacity>
        </View>
      </View>
      <View style={styles.imageWrap}>
        {item.imageUri ? (
          <Image source={{ uri: item.imageUri }} style={styles.image} resizeMode="cover" />
        ) : (
          <View style={styles.placeholder}>
            <Text style={{ color: "#666", textAlign: "center" }}>لا توجد صورة</Text>
          </View>
        )}
      </View>
    </View>
  );

  return (
    <View style={styles.container}>
      <View style={styles.header}>
        <Text style={styles.headerText}>قائمة 50 مود</Text>
        <TouchableOpacity onPress={exportModsJson}>
          <Text style={styles.export}>تصدير JSON</Text>
        </TouchableOpacity>
      </View>
      <FlatList data={mods} keyExtractor={(i) => i.id} renderItem={renderItem} contentContainerStyle={{ paddingBottom: 40 }} />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, paddingTop: 40, backgroundColor: "#f5f5f5" },
  header: { flexDirection: "row", justifyContent: "space-between", alignItems: "center", paddingHorizontal: 16, marginBottom: 8 },
  headerText: { fontSize: 18, fontWeight: "600" },
  export: { color: "#007aff" },
  card: { flexDirection: "row", padding: 12, marginHorizontal: 12, marginVertical: 6, backgroundColor: "#fff", borderRadius: 8, elevation: 2 },
  title: { fontSize: 16, fontWeight: "600" },
  desc: { fontSize: 13, color: "#444", marginTop: 4 },
  button: { paddingVertical: 6, paddingHorizontal: 10, backgroundColor: "#007aff", borderRadius: 6 },
  btnText: { color: "#fff" },
  imageWrap: { width: 90, height: 90, marginLeft: 12, borderRadius: 6, overflow: "hidden", backgroundColor: "#eee" },
  image: { width: "100%", height: "100%" },
  placeholder: { flex: 1, justifyContent: "center", alignItems: "center" },
});
