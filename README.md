# fblite
tool fblite





import re
import tkinter as tk
from tkinter import ttk, messagebox, scrolledtext
from ppadb.client import Client as AdbClient
from threading import Thread

class FBLinkOpenerPro:
    def __init__(self, root):
        self.root = root
        root.title("Facebook Lite Link Opener Pro")
        root.geometry("750x650")

        # Khởi tạo giao diện
        self.create_widgets()
        self.refresh_devices()

    def create_widgets(self):
        # Frame thiết bị
        device_frame = ttk.LabelFrame(self.root, text="📱 Thiết bị đang kết nối")
        device_frame.pack(pady=10, padx=10, fill="both", expand=True)

        # Listbox thiết bị
        self.device_list = tk.Listbox(
            device_frame,
            selectmode=tk.MULTIPLE,
            height=8,
            font=('Arial', 10),
            bg="#f0f0f0"
        )
        self.device_list.pack(side="left", fill="both", expand=True)

        # Thanh cuộn
        scrollbar = ttk.Scrollbar(device_frame)
        scrollbar.pack(side="right", fill="y")
        self.device_list.config(yscrollcommand=scrollbar.set)
        scrollbar.config(command=self.device_list.yview)

        # Nút làm mới
        self.refresh_btn = ttk.Button(
            self.root,
            text="🔄 Làm mới thiết bị",
            command=self.refresh_devices
        )
        self.refresh_btn.pack(pady=5)

        # Frame chọn loại liên kết
        type_frame = ttk.LabelFrame(self.root, text="🔗 Chọn loại thao tác")
        type_frame.pack(pady=5, fill="x", padx=10)

        self.action_type = tk.StringVar(value="profile")
        ttk.Radiobutton(
            type_frame,
            text="👤 Trang cá nhân",
            variable=self.action_type,
            value="profile",
            command=self.toggle_input_fields
        ).pack(side="left", padx=10)
        
        ttk.Radiobutton(
            type_frame,
            text="👥 Nhóm",
            variable=self.action_type,
            value="group",
            command=self.toggle_input_fields
        ).pack(side="left", padx=10)
        
        ttk.Radiobutton(
            type_frame,
            text="📄 Bài viết trong nhóm",
            variable=self.action_type,
            value="post",
            command=self.toggle_input_fields
        ).pack(side="left", padx=10)

        # Frame nhập liệu
        self.input_frame = ttk.Frame(self.root)
        self.input_frame.pack(pady=10, fill="x", padx=10)

        # Ô nhập Profile UID
        self.profile_entry = ttk.Entry(self.input_frame, width=60)
        ttk.Label(self.input_frame, text="🔢 UID Profile:").pack(side="left")
        self.profile_entry.pack(side="left", padx=5)

        # Ô nhập Group UID/URL
        self.group_uid_entry = ttk.Entry(self.input_frame, width=60)
        self.group_uid_label = ttk.Label(self.input_frame, text="🔢 Group UID/URL:")
        
        # Ô nhập Post Link
        self.post_link_entry = ttk.Entry(self.input_frame, width=60)
        self.post_link_label = ttk.Label(self.input_frame, text="🔗 Post Link:")

        # Nút thực thi
        self.execute_btn = ttk.Button(
            self.root,
            text="🚀 Thực thi",
            command=self.start_execution,
            style="Accent.TButton"
        )
        self.execute_btn.pack(pady=10)

        # Console log
        self.console = scrolledtext.ScrolledText(
            self.root,
            height=12,
            wrap=tk.WORD,
            state='disabled',
            font=('Consolas', 9)
        )
        self.console.pack(pady=10, padx=10, fill="both", expand=True)

        # Ẩn các ô nhập không cần thiết ban đầu
        self.toggle_input_fields()

        # Style
        style = ttk.Style()
        style.configure("Accent.TButton", foreground="white", background="#0078d4")

    def toggle_input_fields(self):
        # Xóa tất cả các widget trong frame
        for widget in self.input_frame.winfo_children():
            widget.destroy()

        action_type = self.action_type.get()
        
        if action_type == "profile":
            ttk.Label(self.input_frame, text="🔢 UID Profile:").pack(side="left")
            self.profile_entry = ttk.Entry(self.input_frame, width=60)
            self.profile_entry.pack(side="left", padx=5)
        elif action_type == "group":
            ttk.Label(self.input_frame, text="🔢 Group UID/URL:").pack(side="left")
            self.group_uid_entry = ttk.Entry(self.input_frame, width=60)
            self.group_uid_entry.pack(side="left", padx=5)
        else:
            ttk.Label(self.input_frame, text="🔗 Post Link:").pack(side="left")
            self.post_link_entry = ttk.Entry(self.input_frame, width=60)
            self.post_link_entry.pack(side="left", padx=5)

    def refresh_devices(self):
        self.device_list.delete(0, tk.END)
        try:
            client = AdbClient(host="127.0.0.1", port=5037)
            self.devices = client.devices()
            for device in self.devices:
                model = device.shell("getprop ro.product.model").strip() or "Unknown"
                self.device_list.insert(tk.END, f"{model} ({device.serial})")
            self.log_message("✅ Đã cập nhật danh sách thiết bị")
        except Exception as e:
            messagebox.showerror("Lỗi", f"🔌 Kết nối ADB thất bại: {str(e)}")

    def log_message(self, message):
        self.console.config(state='normal')
        self.console.insert(tk.END, message + "\n")
        self.console.see(tk.END)
        self.console.config(state='disabled')

    def extract_post_info(self, input_str):
        # Pattern cho bài viết trong nhóm
        post_pattern = r"(?:https?:\/\/)?(?:www\.|m\.)?facebook\.com\/groups\/(\d+)\/permalink\/(\d+)"
        post_match = re.search(post_pattern, input_str)
        if post_match:
            return post_match.groups()  # (group_id, post_id)
        
        # Pattern cho ID bài viết dạng số (groupid_postid)
        if re.match(r"^\d+_\d+$", input_str):
            return input_str.split('_')
        
        return None, None

    def extract_group_id(self, input_str):
        # Nếu người dùng nhập URL, trích xuất phần sau /groups/
        if input_str.startswith("http"):
            pattern = r"(?:https?:\/\/)?(?:www\.|m\.)?facebook\.com\/groups\/([^\/\?]+)"
            match = re.search(pattern, input_str)
            if match:
                return match.group(1)
            return None
        else:
            # Nếu nhập trực tiếp UID hoặc tên nhóm
            return input_str.strip()

    def open_link(self, device):
        FB_LITE_PACKAGE = "com.facebook.lite"
        action_type = self.action_type.get()

        try:
            if action_type == "profile":
                profile_id = self.profile_entry.get().strip()
                if not profile_id:
                    self.log_message("❌ Vui lòng nhập UID Profile")
                    return False
                
                profile_id = re.sub(r"\D", "", profile_id)
                url = f"fb://profile/{profile_id}"
                result = device.shell(f'am start -a android.intent.action.VIEW -d "{url}" {FB_LITE_PACKAGE}')
                if "Error" in result:
                    raise Exception(result.split("Error:")[-1])
                self.log_message(f"👤 Đã mở trang cá nhân: {profile_id}")
                return True

            elif action_type == "group":
                group_input = self.group_uid_entry.get().strip()
                if not group_input:
                    self.log_message("❌ Vui lòng nhập Group UID/URL")
                    return False
                
                group_id = self.extract_group_id(group_input)
                if not group_id:
                    self.log_message("❌ Group UID/URL không hợp lệ")
                    return False

                # Thử mở nhóm bằng intent URI
                deep_link = f"intent://groups/{group_id}#Intent;scheme=fb;package={FB_LITE_PACKAGE};end"
                result = device.shell(f'am start -a android.intent.action.VIEW -d "{deep_link}"')
                if "Error" not in result:
                    self.log_message(f"👥 Đã mở nhóm qua intent deep link: {group_id}")
                    return True

                # Fallback sang mở qua URL web nếu intent không thành công
                web_url = f"https://m.facebook.com/groups/{group_id}/"
                result = device.shell(f'am start -a android.intent.action.VIEW -d "{web_url}"')
                if "Error" in result:
                    raise Exception(result.split("Error:")[-1])
                self.log_message(f"🌐 Đang mở nhóm qua web: {group_id}")
                return True

            else:  # post
                post_link = self.post_link_entry.get().strip()
                if not post_link:
                    self.log_message("❌ Vui lòng nhập link bài viết")
                    return False
                
                group_id, post_id = self.extract_post_info(post_link)
                if not group_id or not post_id:
                    self.log_message("❌ Link bài viết không hợp lệ")
                    return False
                
                web_url = f"https://m.facebook.com/groups/{group_id}/posts/{post_id}/"
                result = device.shell(f'am start -a android.intent.action.VIEW -d "{web_url}"')
                if "Error" in result:
                    raise Exception(result.split("Error:")[-1])
                self.log_message(f"📄 Đang mở bài viết {post_id} trong nhóm {group_id}")
                return True

        except Exception as e:
            self.log_message(f"⛔ Lỗi: {str(e)}")
            return False

    def start_execution(self):
        if not hasattr(self, 'devices') or not self.devices:
            messagebox.showwarning("Cảnh báo", "📵 Không tìm thấy thiết bị!")
            return

        selected_indices = self.device_list.curselection()
        if not selected_indices:
            messagebox.showwarning("Cảnh báo", "📱 Vui lòng chọn ít nhất 1 thiết bị!")
            return

        Thread(target=self.execute_commands, args=(selected_indices,), daemon=True).start()

    def execute_commands(self, selected_indices):
        self.refresh_btn.config(state='disabled')
        self.execute_btn.config(state='disabled')
        
        try:
            for index in selected_indices:
                device = self.devices[index]
                self.log_message(f"\n🔧 Đang xử lý {device.serial}...")
                success = self.open_link(device)
                
                if success:
                    self.log_message("✅ Thành công!")
                else:
                    self.log_message("❌ Thất bại")
            
            messagebox.showinfo("Hoàn tất", "🎉 Đã thực hiện xong trên các thiết bị đã chọn!")
        except Exception as e:
            messagebox.showerror("Lỗi", f"💥 Lỗi hệ thống: {str(e)}")
        finally:
            self.refresh_btn.config(state='normal')
            self.execute_btn.config(state='normal')

if __name__ == "__main__":
    root = tk.Tk()
    app = FBLinkOpenerPro(root)
    root.mainloop()
