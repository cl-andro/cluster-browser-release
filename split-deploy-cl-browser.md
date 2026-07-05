# Deploying Cluster Browser Unsigned APK (Split Method)

This guide documents the split-chunk deployment method used to upload the compiled unsigned APK of **Cluster Browser** to your private repository (`cl-andro/cluster-browser`). 

This approach is necessary because the compiled Chromium/Cromite APK is approximately **186MB**, which exceeds GitHub's default non-LFS file upload limit of **100MB**.

---

## 📋 Pre-requisites

*   You have compiled the Chromium project and have a fresh unsigned build at:
    `/home/alamgir-zk/build/chromium/src/out/arm64/apks/ChromePublic.apk`
*   Your local repository `Cluster-Browser01` is located at:
    `/home/alamgir-zk/Cluster-Family/Cluster-Browser01`

---

## 🚀 Deployment Steps (Local Machine)

Run the following commands on your local machine to split the APK and push it to the private repository:

### 1. Remove Any Previously Signed APK (To force a clean compile if needed)
If you need to make sure the latest modifications are fully rebuilt as unsigned:
```bash
rm -f /home/alamgir-zk/build/chromium/src/out/arm64/apks/ChromePublic.apk
```

### 2. Compile/Package the Unsigned APK
Export `depot_tools` and package the public Chromium APK:
```bash
export PATH="/home/alamgir-zk/build/chromium/src/third_party/depot_tools:$PATH"
ninja -j12 -C /home/alamgir-zk/build/chromium/src/out/arm64 chrome_public_apk
```

### 3. Split the Unsigned APK into Parts Under 100MB
Use the standard Linux `split` utility to split the `186MB` APK into chunks of **`90MB`**:
```bash
split -b 90M \
  /home/alamgir-zk/build/chromium/src/out/arm64/apks/ChromePublic.apk \
  /home/alamgir-zk/Cluster-Family/Cluster-Browser01/cluster-browser-unsigned.apk.part.
```
This generates three files under `Cluster-Browser01`:
*   `cluster-browser-unsigned.apk.part.aa` (~90MB)
*   `cluster-browser-unsigned.apk.part.ab` (~90MB)
*   `cluster-browser-unsigned.apk.part.ac` (~6MB)

### 4. Push the Split Chunks to GitHub
Commit and push the chunks to your private browser repository:
```bash
cd /home/alamgir-zk/Cluster-Family/Cluster-Browser01
git add cluster-browser-unsigned.apk.part.aa cluster-browser-unsigned.apk.part.ab cluster-browser-unsigned.apk.part.ac
git commit -m "Release: Add split unsigned APK parts for GitHub Actions release workflow"
git push origin main
```

---

## 🛠️ Reconstruction Step (GitHub Actions Runner)

In your GitHub Actions workflow (`.github/workflows/cluster-browser-apk.yml`), the runner concatenates the split parts back into the complete `unsigned.apk` file using the `cat` command:

```yaml
      - name: Reconstruct Unsigned APK from Split Parts
        run: |
          if [ ! -f browser-source/cluster-browser-unsigned.apk.part.aa ]; then
            echo "Error: Split parts for unsigned APK not found in the private repository!"
            exit 1
          fi
          cat browser-source/cluster-browser-unsigned.apk.part.* > ./unsigned.apk
          echo "Reconstructed unsigned APK successfully."
```

Once reconstructed, the workflow proceeds to run standard `zipalign` and `apksigner` steps to sign and release the final binary.
