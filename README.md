import sys
import sqlite3
from PyQt6.QtWidgets import (
    QApplication, QMainWindow, QWidget, QPushButton, QVBoxLayout, QLabel,
    QTableWidget, QTableWidgetItem, QLineEdit, QDialog, QFormLayout,
    QHBoxLayout, QMessageBox
)
from PyQt6.QtCore import Qt

DB_NAME = "catalogos.db"

class DatabaseManager:
    """Manejo de la base de datos SQLite."""
    def __init__(self):
        self.initialize_database()

    def initialize_database(self):
        """Inicializa la base de datos y crea tablas si no existen."""
        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()

        cursor.execute("""
            CREATE TABLE IF NOT EXISTS articulo (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                nombre TEXT NOT NULL,
                precio REAL NOT NULL
            )
        """)

        cursor.execute("""
            CREATE TABLE IF NOT EXISTS cliente (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                nombre TEXT NOT NULL
            )
        """)

        cursor.execute("""
            CREATE TABLE IF NOT EXISTS empleado (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                nombre TEXT NOT NULL
            )
        """)

        conn.commit()
        conn.close()

class AddDialog(QDialog):
    """Ventana para registrar productos, clientes y empleados."""
    def __init__(self, table_name, main_window):
        super().__init__()
        self.table_name = table_name
        self.main_window = main_window
        self.setWindowTitle(f"Agregar {table_name.capitalize()}")
        self.setup_ui()

    def setup_ui(self):
        layout = QFormLayout()

        self.input_nombre = QLineEdit()
        self.input_nombre.setPlaceholderText("Ingrese nombre")
        layout.addRow("Nombre:", self.input_nombre)

        if self.table_name == "articulo":
            self.input_precio = QLineEdit()
            self.input_precio.setPlaceholderText("Ingrese precio")
            layout.addRow("Precio:", self.input_precio)

        btn_guardar = QPushButton("Guardar")
        btn_guardar.setStyleSheet("background-color: #2b5288; color: white; padding: 10px;")
        btn_guardar.clicked.connect(self.save_data)
        layout.addWidget(btn_guardar)

        self.setLayout(layout)

    def save_data(self):
        nombre = self.input_nombre.text().strip()
        if not nombre:
            QMessageBox.warning(self, "Error", "El nombre no puede estar vacío.")
            return

        conn = sqlite3.connect(DB_NAME)
        cursor = conn.cursor()

        try:
            if self.table_name == "articulo":
                try:
                    precio = float(self.input_precio.text())
                except ValueError:
                    QMessageBox.warning(self, "Error", "El precio debe ser un número válido.")
                    return
                cursor.execute("INSERT INTO articulo (nombre, precio) VALUES (?, ?)", (nombre, precio))
            else:
                cursor.execute(f"INSERT INTO {self.table_name} (nombre) VALUES (?)", (nombre,))
            conn.commit()
            QMessageBox.information(self, "Éxito", f"{self.table_name.capitalize()} registrado correctamente.")
            self.main_window.load_all_tables()
            self.accept()
        except sqlite3.Error as e:
            QMessageBox.warning(self, "Error", f"No se pudo guardar el registro: {e}")
        finally:
            conn.close()

class MainWindow(QMainWindow):
    """Ventana principal con el sistema de gestión de Playeras El Tilín."""
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Playeras El Tilín - Sistema de Gestión")
        self.setGeometry(200, 200, 1000, 700)
        self.setup_ui()

    def setup_ui(self):
        main_widget = QWidget()
        main_layout = QHBoxLayout(main_widget)

        menu_layout = QVBoxLayout()
        menu_layout.setAlignment(Qt.AlignmentFlag.AlignTop)

        botones = {
            "Registrar Producto": lambda: self.open_add_dialog("articulo"),
            "Registrar Cliente": lambda: self.open_add_dialog("cliente"),
            "Registrar Empleado": lambda: self.open_add_dialog("empleado"),
            "Actualizar Tablas": self.load_all_tables,
            "Salir": self.close
        }

        for nombre, metodo in botones.items():
            btn = QPushButton(nombre)
            btn.setStyleSheet("background-color: #2b5288; color: white; font-size: 16px; padding: 10px; border-radius: 5px;")
            btn.clicked.connect(metodo)
            menu_layout.addWidget(btn)

        menu_widget = QWidget()
        menu_widget.setLayout(menu_layout)
        menu_widget.setFixedWidth(250)

        self.content_layout = QVBoxLayout()
        self.tables = {}
        for table in ["articulo", "cliente", "empleado"]:
            label = QLabel(f"Tabla de {table.capitalize()}")
            label.setStyleSheet("font-size: 18px; font-weight: bold; color: #2b5288;")
            self.content_layout.addWidget(label)
            table_widget = QTableWidget()
            self.tables[table] = table_widget
            self.content_layout.addWidget(table_widget)

            delete_btn = QPushButton(f"Eliminar {table.capitalize()} Seleccionado")
            delete_btn.setStyleSheet("background-color: #4a4a4a; color: white; font-size: 14px; padding: 8px; border-radius: 5px;")
            delete_btn.clicked.connect(lambda _, t=table: self.delete_selected(t))
            self.content_layout.addWidget(delete_btn)

        content_widget = QWidget()
        content_widget.setLayout(self.content_layout)

        main_layout.addWidget(menu_widget)
        main_layout.addWidget(content_widget)
        self.setCentralWidget(main_widget)
        self.load_all_tables()

    def open_add_dialog(self, table_name):
        """Abre la ventana de registro según la tabla seleccionada."""
        dialog = AddDialog(table_name, self)
        dialog.exec()

    def delete_selected(self, table_name):
        """Elimina el registro seleccionado en la tabla."""
        table = self.tables[table_name]
        selected_row = table.currentRow()
        if selected_row < 0:
            QMessageBox.warning(self, "Error", f"Seleccione un {table_name} para eliminar.")
            return

        id_item = table.item(selected_row, 0)
        if not id_item:
            return

        record_id = id_item.text()

        confirm = QMessageBox.question(
            self, "Confirmar eliminación",
            f"¿Seguro que deseas eliminar este {table_name}?",
            QMessageBox.StandardButton.Yes | QMessageBox.StandardButton.No
        )

        if confirm == QMessageBox.StandardButton.Yes:
            conn = sqlite3.connect(DB_NAME)
            cursor = conn.cursor()
            try:
                cursor.execute(f"DELETE FROM {table_name} WHERE id = ?", (record_id,))
                conn.commit()
            except sqlite3.Error as e:
                QMessageBox.warning(self, "Error", f"No se pudo eliminar: {e}")
            finally:
                conn.close()
            self.load_all_tables()

if __name__ == "__main__":
    db_manager = DatabaseManager()
    app = QApplication(sys.argv)
    window = MainWindow()
    window.show()
    sys.exit(app.exec())
