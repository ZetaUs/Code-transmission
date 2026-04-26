# Code

## FileManager.py
```
import os
import shutil
from PyQt5.QtWidgets import (QApplication, QMainWindow, QWidget, QHBoxLayout, 
                             QVBoxLayout, QListWidget, QListWidgetItem, QTreeWidget,
                             QTreeWidgetItem, QLineEdit, QToolBar, QAction, 
                             QMenu, QMessageBox, QFileDialog, QLabel)
from PyQt5.QtCore import Qt, QDir
from PyQt5.QtGui import QIcon, QFont

class ExquisiteFileManager(QMainWindow):
    def __init__(self):
        super().__init__()
        self.current_path = os.path.expanduser("~")
        self.clipboard_path = None
        self.clipboard_op = None  # copy / cut
        self.initUI()

    def initUI(self):
        self.setWindowTitle("精致文件管理器")
        self.resize(1200, 750)
        self.setStyleSheet("""
        QMainWindow {background: #1e2126;}
        QLineEdit {background: #2c313a; color: #dcdfe4; border: none; padding: 6px; border-radius: 4px; font-size:14px;}
        QListWidget, QTreeWidget {background: #252930; color: #dcdfe4; border: none; outline:none;}
        QListWidget::item:hover, QTreeWidget::item:hover {background:#373c45;}
        QListWidget::item:selected, QTreeWidget::item:selected {background:#414853;}
        QToolBar {background:#252930; border:none; padding:4px;}
        QMenu {background:#2c313a; color:#dcdfe4; border:1px solid #414853;}
        QMenu::item:selected {background:#373c45;}
        """)

        # 顶部地址栏
        toolbar = QToolBar()
        self.addToolBar(toolbar)
        self.address_edit = QLineEdit()
        self.address_edit.setText(self.current_path)
        self.address_edit.returnPressed.connect(self.go_path)
        toolbar.addWidget(self.address_edit)

        # 主布局：左侧边栏 + 右文件列表
        main_widget = QWidget()
        self.setCentralWidget(main_widget)
        main_layout = QHBoxLayout(main_widget)
        main_layout.setContentsMargins(5,5,5,5)
        main_layout.setSpacing(5)

        # 左侧快捷目录
        self.side_tree = QTreeWidget()
        self.side_tree.setFixedWidth(220)
        self.side_tree.setHeaderHidden(True)
        self.load_side_dirs()
        self.side_tree.itemClicked.connect(self.side_click)

        # 右侧文件列表
        self.file_list = QListWidget()
        self.file_list.setViewMode(QListWidget.ListMode)
        self.file_list.setResizeMode(QListWidget.Adjust)
        self.file_list.doubleClicked.connect(self.file_double_click)
        self.file_list.setContextMenuPolicy(Qt.CustomContextMenu)
        self.file_list.customContextMenuRequested.connect(self.show_right_menu)

        main_layout.addWidget(self.side_tree)
        main_layout.addWidget(self.file_list)

        # 加载初始目录
        self.load_dir(self.current_path)

    def load_side_dirs(self):
        root = QTreeWidgetItem(self.side_tree, ["常用位置"])
        home = os.path.expanduser("~")
        paths = [
            ("桌面", os.path.join(home, "Desktop")),
            ("文档", os.path.join(home, "Documents")),
            ("下载", os.path.join(home, "Downloads")),
            ("图片", os.path.join(home, "Pictures")),
            ("视频", os.path.join(home, "Videos")),
            ("本地磁盘 C", "C:\\"),
            ("本地磁盘 D", "D:\\")
        ]
        for name, path in paths:
            if os.path.exists(path):
                QTreeWidgetItem(root, [name]).setData(0, Qt.UserRole, path)
        root.setExpanded(True)

    def side_click(self, item):
        path = item.data(0, Qt.UserRole)
        if path and os.path.exists(path):
            self.load_dir(path)
            self.address_edit.setText(path)

    def go_path(self):
        path = self.address_edit.text().strip()
        if os.path.exists(path):
            self.load_dir(path)
        else:
            QMessageBox.warning(self, "提示", "路径不存在！")

    def load_dir(self, path):
        self.current_path = path
        self.address_edit.setText(path)
        self.file_list.clear()
        try:
            dirs = []
            files = []
            for name in os.listdir(path):
                full = os.path.join(path, name)
                if os.path.isdir(full):
                    dirs.append(name)
                else:
                    files.append(name)
            # 先文件夹后文件
            for name in sorted(dirs):
                item = QListWidgetItem("📁 " + name)
                item.setData(Qt.UserRole, os.path.join(path, name))
                self.file_list.addItem(item)
            for name in sorted(files):
                item = QListWidgetItem("📄 " + name)
                item.setData(Qt.UserRole, os.path.join(path, name))
                self.file_list.addItem(item)
        except Exception as e:
            QMessageBox.warning(self, "访问失败", f"无法访问目录：{str(e)}")

    def file_double_click(self):
        item = self.file_list.currentItem()
        if not item:
            return
        path = item.data(Qt.UserRole)
        if os.path.isdir(path):
            self.load_dir(path)
        else:
            # 用系统默认程序打开文件
            if os.name == "nt":
                os.startfile(path)
            else:
                os.system(f"xdg-open '{path}'")

    def show_right_menu(self, pos):
        menu = QMenu()
        open_act = menu.addAction("打开")
        new_folder = menu.addAction("新建文件夹")
        new_file = menu.addAction("新建文件")
        menu.addSeparator()
        copy_act = menu.addAction("复制")
        cut_act = menu.addAction("剪切")
        paste_act = menu.addAction("粘贴")
        menu.addSeparator()
        rename_act = menu.addAction("重命名")
        del_act = menu.addAction("删除")

        action = menu.exec_(self.file_list.mapToGlobal(pos))
        item = self.file_list.currentItem()
        item_path = item.data(Qt.UserRole) if item else None

        if action == open_act and item_path:
            self.file_double_click()
        elif action == new_folder:
            self.mkdir_dialog()
        elif action == new_file:
            self.mkfile_dialog()
        elif action == copy_act and item_path:
            self.clipboard_path = item_path
            self.clipboard_op = "copy"
        elif action == cut_act and item_path:
            self.clipboard_path = item_path
            self.clipboard_op = "cut"
        elif action == paste_act and self.clipboard_path:
            self.paste_file()
        elif action == rename_act and item_path:
            self.rename_dialog(item_path)
        elif action == del_act and item_path:
            self.delete_file(item_path)

    def mkdir_dialog(self):
        name, ok = QFileDialog.getSaveFileName(self, "新建文件夹", self.current_path, "")
        if ok and name:
            try:
                os.mkdir(name)
                self.load_dir(self.current_path)
            except Exception as e:
                QMessageBox.warning(self, "错误", str(e))

    def mkfile_dialog(self):
        name, ok = QFileDialog.getSaveFileName(self, "新建文件", self.current_path)
        if ok and name:
            try:
                with open(name, "w", encoding="utf-8") as f:
                    pass
                self.load_dir(self.current_path)
            except Exception as e:
                QMessageBox.warning(self, "错误", str(e))

    def paste_file(self):
        dst = self.current_path
        src = self.clipboard_path
        try:
            new_path = os.path.join(dst, os.path.basename(src))
            # 处理重名
            if os.path.exists(new_path):
                QMessageBox.warning(self, "提示", "目标位置已存在同名文件/文件夹")
                return
            if os.path.isdir(src):
                shutil.copytree(src, new_path)
                if self.clipboard_op == "cut":
                    shutil.rmtree(src)
            else:
                shutil.copy2(src, new_path)
                if self.clipboard_op == "cut":
                    os.remove(src)
            # 剪切后清空剪贴板
            if self.clipboard_op == "cut":
                self.clipboard_path = None
            self.load_dir(dst)
        except Exception as e:
            QMessageBox.warning(self, "粘贴失败", str(e))

    def rename_dialog(self, old_path):
        base = os.path.dirname(old_path)
        old_name = os.path.basename(old_path)
        new_name, ok = QFileDialog.getSaveFileName(self, "重命名", os.path.join(base, old_name))
        if ok and new_name:
            try:
                os.rename(old_path, new_name)
                self.load_dir(self.current_path)
            except Exception as e:
                QMessageBox.warning(self, "重命名失败", str(e))

    def delete_file(self, path):
        reply = QMessageBox.question(self, "确认删除", "确定要删除该文件/文件夹？无法恢复！",
                                     QMessageBox.Yes | QMessageBox.No)
        if reply == QMessageBox.Yes:
            try:
                if os.path.isdir(path):
                    shutil.rmtree(path)
                else:
                    os.remove(path)
                self.load_dir(self.current_path)
            except Exception as e:
                QMessageBox.warning(self, "删除失败", str(e))

if __name__ == "__main__":
    import sys
    app = QApplication(sys.argv)
    win = ExquisiteFileManager()
    win.show()
    sys.exit(app.exec_())
```

## Windows box.py
```
import os
import subprocess

def create_and_launch_sandbox():
    # 1. 定义 .wsb 配置文件的内容（XML 格式）
    # 你可以在这里像搭积木一样，定制你的专属沙盒环境
    wsb_content = """<Configuration>
  <!-- 启用虚拟GPU，让沙盒里的界面更流畅 -->
  <VGpu>Enable</VGpu>
  
  <!-- 禁用网络！打造一个绝对断网的单机测试环境 -->
  <Networking>Disable</Networking>
  
  <!-- 映射主机文件夹：把你电脑上的文件夹“刻录”进沙盒 -->
  <MappedFolders>
    <MappedFolder>
      <!-- 这里写你电脑上真实存在的文件夹路径 -->
      <HostFolder>C:\\Users\\Public\\Documents</HostFolder>
      <!-- 设为 true 表示沙盒里只能看不能改，保护你的原文件 -->
      <ReadOnly>true</ReadOnly>
    </MappedFolder>
  </MappedFolders>

  <!-- 登录命令：沙盒启动后，自动帮你打开记事本 -->
  <LogonCommand>
    <Command>notepad.exe</Command>
  </LogonCommand>
</Configuration>
"""

    # 2. 将配置内容“刻录”成一个真实的 .wsb 文件
    wsb_filename = "My_Custom_Sandbox.wsb"
    print(f"--- 🛠️ 正在刻录专属沙盒配置文件: {wsb_filename} ---")
    with open(wsb_filename, "w", encoding="utf-8") as f:
        f.write(wsb_content)
    
    # 3. 指挥 Windows 启动这个定制好的沙盒
    print("--- 🚀 正在启动你的专属 Windows 沙盒... ---")
    try:
        # 使用 os.startfile 可以直接双击打开 .wsb 文件
        os.startfile(wsb_filename)
        print("✅ 专属沙盒已成功弹出！你可以去里面随意折腾了。")
    except Exception as e:
        print(f"⛔ 启动失败，请检查你的电脑是否已开启 Windows 沙盒功能。\n报错: {e}")

if __name__ == "__main__":
    create_and_launch_sandbox()
```

## code_anim.py
```
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import Dataset, DataLoader
import pygame
import sys
import numpy as np

# ---------------------- 1. 数据与模型部分 ----------------------
# 构建文本数据
texts = [
    "hello world",
    "hello yellow",
    "coding is fun",
    "python is cool",
    "animation vs code"
]

# 构建词汇表
vocab = set()
for text in texts:
    vocab.update(text.split())
vocab = sorted(vocab)
word_to_idx = {word: idx for idx, word in enumerate(vocab)}
idx_to_word = {idx: word for word, idx in word_to_idx.items()}
vocab_size = len(vocab)

seq_length = 2  # 用前2个词预测第3个

# 文本数据集类
class TextDataset(Dataset):
    def __init__(self, texts, word_to_idx, seq_length=2):
        self.texts = texts
        self.word_to_idx = word_to_idx
        self.seq_length = seq_length
        self.data = []
        for text in texts:
            tokens = text.split()
            # 生成 (前n个词, 下一个词)
            for i in range(len(tokens) - seq_length):
                x = tokens[i:i+seq_length]
                y = tokens[i+seq_length]
                self.data.append((x, y))
    
    def __len__(self):
        return len(self.data)
    
    def __getitem__(self, idx):
        x_tokens, y_token = self.data[idx]
        x = torch.tensor([self.word_to_idx[tok] for tok in x_tokens], dtype=torch.long)
        y = torch.tensor(self.word_to_idx[y_token], dtype=torch.long)
        return x, y

# LSTM 模型
class LSTMModel(nn.Module):
    def __init__(self, vocab_size, embedding_dim=16, hidden_dim=32, seq_length=2):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embedding_dim)
        self.lstm = nn.LSTM(embedding_dim, hidden_dim, batch_first=True)
        self.fc = nn.Linear(hidden_dim, vocab_size)
    
    def forward(self, x):
        embedded = self.embedding(x)
        lstm_out, _ = self.lstm(embedded)
        out = self.fc(lstm_out[:, -1, :])  # 取最后一步输出
        return out

# 初始化
dataset = TextDataset(texts, word_to_idx, seq_length)
dataloader = DataLoader(dataset, batch_size=2, shuffle=True)
model = LSTMModel(vocab_size)
criterion = nn.CrossEntropyLoss()
optimizer = optim.Adam(model.parameters(), lr=0.01)

# 训练
epochs = 100
for epoch in range(epochs):
    total_loss = 0
    for x, y in dataloader:
        optimizer.zero_grad()
        output = model(x)
        loss = criterion(output, y)
        loss.backward()
        optimizer.step()
        total_loss += loss.item()
    if (epoch + 1) % 10 == 0:
        avg_loss = total_loss / len(dataloader)
        print(f"Epoch {epoch+1:3d} | Loss: {avg_loss:.4f}")

# 预测函数（自动处理长度）
def predict_next_word(input_sequence, model, word_to_idx, idx_to_word, seq_len=2):
    tokens = input_sequence.strip().split()
    # 只取最后 seq_len 个词
    tokens = tokens[-seq_len:]
    # 如果不够，直接返回空
    if len(tokens) != seq_len:
        return "need more words"
    
    try:
        x = torch.tensor([[word_to_idx[w] for w in tokens]], dtype=torch.long)
        with torch.no_grad():
            out = model(x)
            idx = torch.argmax(out).item()
        return idx_to_word[idx]
    except KeyError as e:
        return f"unknown word: {str(e)}"

# ---------------------- 2. Pygame 可视化交互 ----------------------
pygame.init()
W, H = 800, 600
screen = pygame.display.set_mode((W, H))
pygame.display.set_caption("Animation vs Code - LSTM Text Predictor")
font = pygame.font.SysFont("Consolas", 26)
clock = pygame.time.Clock()

input_text = ""
predicted_word = ""

while True:
    screen.fill((0, 0, 0))
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            pygame.quit()
            sys.exit()
        
        elif event.type == pygame.KEYDOWN:
            if event.key == pygame.K_RETURN:
                # 回车预测
                predicted_word = predict_next_word(input_text, model, word_to_idx, idx_to_word, seq_length)
            elif event.key == pygame.K_BACKSPACE:
                input_text = input_text[:-1]
            elif event.key == pygame.K_ESCAPE:
                input_text = ""
                predicted_word = ""
            else:
                # 只允许可见字符
                if event.unicode.isprintable():
                    input_text += event.unicode

    # 绘制
    surf_input = font.render(f"Input: {input_text}", True, (220, 220, 220))
    screen.blit(surf_input, (50, 100))

    if predicted_word:
        color = (0, 255, 100) if "unknown" not in predicted_word and "need" not in predicted_word else (255, 80, 80)
        surf_pred = font.render(f"Predicted: {predicted_word}", True, color)
        screen.blit(surf_pred, (50, 160))

    pygame.display.flip()
    clock.tick(60)
```
### 依赖
```
pip install torch pygame numpy
```
