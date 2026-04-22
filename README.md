# fgjtyk
import tkinter as tk
from tkinter import ttk, messagebox
import json
import os
from datetime import datetime

class ExpenseTracker:
    def __init__(self, root):
        self.root = root
        self.root.title("Expense Tracker")
        self.root.geometry("900x600")

        # Файл для хранения данных
        self.data_file = "expenses_data.json"
        self.expenses = []
        self.load_expenses()

        self.setup_ui()

    def setup_ui(self):
        # Форма добавления расхода
        form_frame = ttk.LabelFrame(self.root, text="Добавить расход")
        form_frame.pack(pady=10, padx=10, fill="x")

        # Сумма
        ttk.Label(form_frame, text="Сумма:").grid(row=0, column=0, sticky="w", padx=5, pady=5)
        self.amount_entry = ttk.Entry(form_frame, width=20)
        self.amount_entry.grid(row=0, column=1, padx=5, pady=5)

        # Категория
        ttk.Label(form_frame, text="Категория:").grid(row=1, column=0, sticky="w", padx=5, pady=5)
        categories = ["Еда", "Транспорт", "Развлечения", "Жильё", "Одежда", "Другое"]
        self.category_var = tk.StringVar()
        self.category_combo = ttk.Combobox(form_frame, textvariable=self.category_var, values=categories, state="readonly")
        self.category_combo.grid(row=1, column=1, padx=5, pady=5)

        # Дата
        ttk.Label(form_frame, text="Дата (ГГГГ-ММ-ДД):").grid(row=2, column=0, sticky="w", padx=5, pady=5)
        self.date_entry = ttk.Entry(form_frame, width=20)
        self.date_entry.insert(0, datetime.now().strftime("%Y-%m-%d"))
        self.date_entry.grid(row=2, column=1, padx=5, pady=5)

        # Кнопка добавления
        add_btn = ttk.Button(form_frame, text="Добавить расход", command=self.add_expense)
        add_btn.grid(row=3, column=0, columnspan=2, pady=10)

        # Фильтры и подсчёт суммы
        filter_frame = ttk.LabelFrame(self.root, text="Фильтры и статистика")
        filter_frame.pack(pady=5, padx=10, fill="x")

        # Фильтр по категории
        ttk.Label(filter_frame, text="Фильтр по категории:").pack(side="left", padx=5)
        self.filter_category_var = tk.StringVar(value="все")
        category_filter = ttk.Combobox(
            filter_frame,
            textvariable=self.filter_category_var,
            values=["все"] + categories,
            state="readonly"
        )
        category_filter.pack(side="left", padx=5)

        # Фильтр по дате
        ttk.Label(filter_frame, text="С:").pack(side="left", padx=5)
        self.start_date_entry = ttk.Entry(filter_frame, width=12)
        self.start_date_entry.insert(0, "2023-01-01")
        self.start_date_entry.pack(side="left", padx=5)

        ttk.Label(filter_frame, text="По:").pack(side="left", padx=5)
        self.end_date_entry = ttk.Entry(filter_frame, width=12)
        self.end_date_entry.insert(0, datetime.now().strftime("%Y-%m-%d"))
        self.end_date_entry.pack(side="left", padx=5)

        # Кнопки фильтров и подсчёта
        apply_filter_btn = ttk.Button(filter_frame, text="Применить фильтр", command=self.apply_filters)
        apply_filter_btn.pack(side="left", padx=5)

        calc_sum_btn = ttk.Button(filter_frame, text="Подсчитать сумму", command=self.calculate_sum)
        calc_sum_btn.pack(side="left", padx=5)

        # Поле отображения суммы
        self.sum_label = ttk.Label(filter_frame, text="Общая сумма: 0 руб.")
        self.sum_label.pack(side="left", padx=20)

        # Таблица расходов
        table_frame = ttk.LabelFrame(self.root, text="Список расходов")
        table_frame.pack(pady=10, padx=10, fill="both", expand=True)

        columns = ("amount", "category", "date", "description")
        self.tree = ttk.Treeview(table_frame, columns=columns, show="headings", height=15)

        self.tree.heading("amount", text="Сумма (руб.)")
        self.tree.heading("category", text="Категория")
        self.tree.heading("date", text="Дата")
        self.tree.heading("description", text="Описание")

        self.tree.column("amount", width=100)
        self.tree.column("category", width=120)
        self.tree.column("date", width=100)
        self.tree.column("description", width=400)

        scrollbar = ttk.Scrollbar(table_frame, orient="vertical", command=self.tree.yview)
        self.tree.configure(yscrollcommand=scrollbar.set)

        self.tree.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")

        # Кнопки сохранения/загрузки
        action_frame = ttk.Frame(self.root)
        action_frame.pack(pady=10)

        save_btn = ttk.Button(action_frame, text="Сохранить в JSON", command=self.save_expenses)
        save_btn.pack(side="left", padx=5)

        load_btn = ttk.Button(action_frame, text="Загрузить из JSON", command=self.load_expenses)
        load_btn.pack(side="left", padx=5)

        delete_btn = ttk.Button(action_frame, text="Удалить выбранный", command=self.delete_expense)
        delete_btn.pack(side="left", padx=5)

        self.refresh_table()

    def validate_input(self):
        """Проверка корректности ввода"""
        amount_str = self.amount_entry.get().strip()
        category = self.category_var.get()
        date_str = self.date_entry.get().strip()

        if not amount_str:
            messagebox.showerror("Ошибка", "Сумма не может быть пустой!")
            return False
        try:
            amount = float(amount_str)
            if amount <= 0:
                messagebox.showerror("Ошибка", "Сумма должна быть положительным числом!")
                return False
        except ValueError:
            messagebox.showerror("Ошибка", "Сумма должна быть числом!")
            return False

        if not category:
            messagebox.showerror("Ошибка", "Выберите категорию!")
            return False

        try:
            date = datetime.strptime(date_str, "%Y-%m-%d")
        except ValueError:
            messagebox.showerror("Ошибка", "Дата должна быть в формате ГГГГ-ММ-ДД!")
            return False

        return True, amount, category, date_str
