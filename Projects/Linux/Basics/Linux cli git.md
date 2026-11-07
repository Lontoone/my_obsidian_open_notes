# First set-up on new device

- **產生新的 SSH Key（建議使用 Ed25519）** 在終端機中執行：      
    
    ```Bash
    ssh-keygen -t ed25519 -C "your_email@example.com"
    ```
    
    - 提示 `Enter file in which to save the key` 時，直接按 **Enter** 使用預設路徑（`~/.ssh/id_ed25519`）。
        
    - 提示 `Enter passphrase` 時，可直接按 **Enter** 跳過密碼，或輸入一組自訂密碼後按 Enter。
        
- **複製公鑰內容** 印出公鑰並完整複製整行字串（以 `ssh-ed25519 AAAAC3...` 開頭）：  
    ```Bash
    cat ~/.ssh/id_ed25519.pub
    ```
    
- **新增公鑰至 GitHub**
    
    - 瀏覽器登入 GitHub，進入 **Settings** → 側邊欄點選 **SSH and GPG keys**。
        
    - 點擊右上角 **New SSH key**。
        
    - **Title**：輸入便於識別的名稱（例如 `Linux-PC`）。
        
    - **Key type**：保持預設 `Authentication Key`。
        
    - **Key**：貼上剛才複製的公鑰字串，最後點擊 **Add SSH key**。
- **設定 `~/.ssh/config` 指定金鑰路徑**- 
	- 執行以下指令建立並編輯設定檔：
	```bash
	nano ~/.ssh/config
	```

	在檔案中貼入以下內容：
	```Plaintext
	Host github.com
	    HostName github.com
	    User git
	    IdentityFile ~/.ssh/forGit
	    IdentitiesOnly yes
	```
	按 `Ctrl + O` 再按 `Enter` 存檔，接著按 `Ctrl + X` 離開。
- **修正金鑰與目錄權限**
	- SSH 對權限檢查非常嚴格，若權限過於開放會直接忽略金鑰：
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config
chmod 600 ~/.ssh/forGit
chmod 644 ~/.ssh/forGit.pub
```
- **測試 SSH 連線** 回到終端機驗證身分認證：    
    ```bash
    ssh -T git@github.com
    ```
    
    _驗證成功訊號：_ 看到類似輸出即代表驗證成功：     
    ```Plaintext
    Hi Lontoone! You've successfully authenticated, but GitHub does not provide shell access.
    ```
    
- **修復專案檔案權限並重新 Push** 因為先前執行過 `sudo git push`，先將專案目錄的擁有者改回目前使用者，再進行推動：        
    
    ```bash
    sudo chown -R $USER:$USER ~/<repo>
    git push
    ```








----

# First set-up on new device

- **Generate a new SSH key (Ed25519 recommended)** Run the following in your terminal:
    
    ```bash
    ssh-keygen -t ed25519 -C "your_email@example.com"
    ```
    
    - When prompted with `Enter file in which to save the key`, press **Enter** to accept the default path (`~/.ssh/id_ed25519`).
        
    - When prompted with `Enter passphrase`, press **Enter** to skip setting a passphrase, or type a custom passphrase and press Enter.
        
- **Copy the public key content** Print the public key and copy the entire string (starts with `ssh-ed25519 AAAAC3...`):
    ```bash
    cat ~/.ssh/id_ed25519.pub
    ```
    
- **Add the public key to GitHub**
    
    - Log in to GitHub in your browser, go to **Settings** → click **SSH and GPG keys** in the sidebar.
        
    - Click **New SSH key** in the top right.
        
    - **Title**: Enter an identifiable name (e.g., `Linux-PC`).
        
    - **Key type**: Keep the default `Authentication Key`.
        
    - **Key**: Paste the copied public key string, then click **Add SSH key**.

- **Configure `~/.ssh/config` to specify the key path**
    - Run the following command to create and edit the config file:
        ```bash
        nano ~/.ssh/config
        ```

        Paste the following contents into the file:
        ```plaintext
        Host github.com
            HostName github.com
            User git
            IdentityFile ~/.ssh/forGit
            IdentitiesOnly yes
        ```
        Press `Ctrl + O` and then `Enter` to save, followed by `Ctrl + X` to exit.

- **Fix key and directory permissions**
    - SSH enforces strict permission checks; if permissions are too open, the key will be ignored:
        ```bash
        chmod 700 ~/.ssh
        chmod 600 ~/.ssh/config
        chmod 600 ~/.ssh/forGit
        chmod 644 ~/.ssh/forGit.pub
        ```

- **Test SSH connection** Return to your terminal to verify authentication:
    ```bash
    ssh -T git@github.com
    ```
    
    _Successful verification:_ Seeing output similar to this indicates success:
    ```plaintext
    Hi Lontoone! You've successfully authenticated, but GitHub does not provide shell access.
    ```
    
- **Fix project file permissions and push again** Because `sudo git push` was previously run, restore ownership of the project directory to the current user before pushing:
    
    ```bash
    sudo chown -R $USER:$USER ~/<repo>
    git push
    ```