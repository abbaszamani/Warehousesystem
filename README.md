# Warehousesystem
package WarehouseSystem;

import javax.swing.*;
import javax.swing.table.DefaultTableModel;
import java.awt.*;
import java.sql.*;
import java.text.SimpleDateFormat;
import java.util.Date;

public class WarehouseGUI {
    private static final String DB_URL = "jdbc:sqlite:warehouse.db";
    private DefaultTableModel model;
    private JTable table;
    private JTextField itemNameField, quantityField, priceField, searchField;
    private JFrame frame;
    private JMenuBar menuBar;

    public WarehouseGUI() {
        createTable();
        createGUI();
        loadProducts();
    }

    private void createTable() {
        String sql = "CREATE TABLE IF NOT EXISTS products (" +
                "id INTEGER PRIMARY KEY AUTOINCREMENT," +
                "code TEXT UNIQUE NOT NULL," +
                "name TEXT NOT NULL," +
                "quantity INTEGER NOT NULL," +
                "price REAL NOT NULL," +
                "entryDate TEXT NOT NULL" +
                ");";
        try (Connection conn = DriverManager.getConnection(DB_URL);
             Statement stmt = conn.createStatement()) {
            stmt.execute(sql);
        } catch (SQLException e) {
            System.out.println("❌ خطا در ایجاد جدول: " + e.getMessage());
        }
    }

    private void loadProducts() {
        model.setRowCount(0);
        String sql = "SELECT * FROM products";
        try (Connection conn = DriverManager.getConnection(DB_URL);
             Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery(sql)) {
            while (rs.next()) {
                model.addRow(new Object[]{
                        rs.getString("code"),
                        rs.getString("name"),
                        rs.getInt("quantity"),
                        rs.getDouble("price"),
                        rs.getString("entryDate")
                });
            }
        } catch (SQLException e) {
            System.out.println("❌ خطا در بارگیری کالاها: " + e.getMessage());
        }
    }

    private void createGUI() {
        frame = new JFrame("مدیریت انبار");
        frame.setSize(900, 700);
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        frame.setLocationRelativeTo(null);

        model = new DefaultTableModel(new String[]{"شناسه", "نام کالا", "تعداد", "قیمت", "تاریخ ورود"}, 0);
        table = new JTable(model);
        JScrollPane scrollPane = new JScrollPane(table);

        JPanel inputPanel = new JPanel(new GridBagLayout());
        GridBagConstraints gbc = new GridBagConstraints();
        gbc.insets = new Insets(5, 5, 5, 5);

        itemNameField = new JTextField(15);
        quantityField = new JTextField(15);
        priceField = new JTextField(15);

        JButton addButton = new JButton("اضافه کردن");
        addButton.addActionListener(e -> addProduct());

        JButton deleteButton = new JButton("حذف کالا");
        deleteButton.addActionListener(e -> deleteProduct());

        searchField = new JTextField(15);
        JButton searchButton = new JButton("جستجو");
        searchButton.addActionListener(e -> searchProduct());

        gbc.gridx = 0;
        gbc.gridy = 0;
        inputPanel.add(new JLabel("نام کالا:"), gbc);
        gbc.gridx = 1;
        inputPanel.add(itemNameField, gbc);
        gbc.gridx = 0;
        gbc.gridy = 1;
        inputPanel.add(new JLabel("تعداد:"), gbc);
        gbc.gridx = 1;
        inputPanel.add(quantityField, gbc);
        gbc.gridx = 0;
        gbc.gridy = 2;
        inputPanel.add(new JLabel("قیمت:"), gbc);
        gbc.gridx = 1;
        inputPanel.add(priceField, gbc);
        gbc.gridx = 0;
        gbc.gridy = 3;
        inputPanel.add(addButton, gbc);
        gbc.gridx = 0;
        gbc.gridy = 4;
        inputPanel.add(deleteButton, gbc);
        gbc.gridx = 0;
        gbc.gridy = 5;
        inputPanel.add(searchField, gbc);
        gbc.gridx = 1;
        gbc.gridy = 5;
        inputPanel.add(searchButton, gbc);

        frame.setLayout(new BorderLayout());
        frame.add(inputPanel, BorderLayout.NORTH);
        frame.add(scrollPane, BorderLayout.CENTER);
        frame.setVisible(true);

        createMenuBar();
    }

    private void createMenuBar() {
        menuBar = new JMenuBar();

        JMenu fileMenu = new JMenu("فایل");
        JMenuItem exitItem = new JMenuItem("خروج");
        exitItem.addActionListener(e -> System.exit(0));
        fileMenu.add(exitItem);

        JMenu helpMenu = new JMenu("راهنما");
        JMenuItem aboutItem = new JMenuItem("درباره");
        aboutItem.addActionListener(e -> JOptionPane.showMessageDialog(frame, "سیستم انبارداری ساده\nنسخه 1.0"));
        helpMenu.add(aboutItem);

        menuBar.add(fileMenu);
        menuBar.add(helpMenu);

        frame.setJMenuBar(menuBar);
    }
}
private void addProduct() {
    String name = itemNameField.getText().trim();
    String quantityText = quantityField.getText().trim();
    String priceText = priceField.getText().trim();

    if (name.isEmpty()  quantityText.isEmpty()  priceText.isEmpty()) {
        JOptionPane.showMessageDialog(frame, "لطفاً تمام فیلدها را پر کنید.", "خطا", JOptionPane.ERROR_MESSAGE);
        return;
    }

    int quantity = Integer.parseInt(quantityText);
    double price = Double.parseDouble(priceText);
    String code = generateProductCode();

    String sql = "INSERT INTO products (code, name, quantity, price, entryDate) VALUES (?, ?, ?, ?, ?)";
    try (Connection conn = DriverManager.getConnection(DB_URL);
         PreparedStatement pstmt = conn.prepareStatement(sql)) {
        pstmt.setString(1, code);
        pstmt.setString(2, name);
        pstmt.setInt(3, quantity);
        pstmt.setDouble(4, price);
        pstmt.setString(5, new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()));
        pstmt.executeUpdate();

        model.addRow(new Object[]{code, name, quantity, price, new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date())});
        JOptionPane.showMessageDialog(frame, "کالا با موفقیت اضافه شد.", "موفقیت", JOptionPane.INFORMATION_MESSAGE);

        itemNameField.setText("");
        quantityField.setText("");
        priceField.setText("");

    } catch (SQLException e) {
        System.out.println("❌ خطا در افزودن کالا: " + e.getMessage());
        JOptionPane.showMessageDialog(frame, "خطا در افزودن کالا به دیتابیس.", "خطا", JOptionPane.ERROR_MESSAGE);
    }
}

private void deleteProduct() {
    int row = table.getSelectedRow();
    if (row == -1) {
        JOptionPane.showMessageDialog(frame, "لطفاً یک کالا را انتخاب کنید.", "خطا", JOptionPane.ERROR_MESSAGE);
        return;
    }

    String code = model.getValueAt(row, 0).toString();
    String sql = "DELETE FROM products WHERE code = ?";
    try (Connection conn = DriverManager.getConnection(DB_URL);
         PreparedStatement pstmt = conn.prepareStatement(sql)) {
        pstmt.setString(1, code);
        pstmt.executeUpdate();
        model.removeRow(row);
        JOptionPane.showMessageDialog(frame, "کالا با موفقیت حذف شد.", "موفقیت", JOptionPane.INFORMATION_MESSAGE);
    } catch (SQLException e) {
        System.out.println("❌ خطا در حذف کالا: " + e.getMessage());
        JOptionPane.showMessageDialog(frame, "خطا در حذف کالا از دیتابیس.", "خطا", JOptionPane.ERROR_MESSAGE);
    }
}

private void searchProduct() {
    String searchQuery = searchField.getText().trim();
    if (searchQuery.isEmpty()) {
        JOptionPane.showMessageDialog(frame, "لطفاً عبارت جستجو را وارد کنید.", "خطا", JOptionPane.ERROR_MESSAGE);
        return;
    }

    String sql = "SELECT * FROM products WHERE name LIKE ?";
    try (Connection conn = DriverManager.getConnection(DB_URL);
         PreparedStatement pstmt = conn.prepareStatement(sql)) {
        pstmt.setString(1, "%" + searchQuery + "%");

        ResultSet rs = pstmt.executeQuery();
        model.setRowCount(0);
        while (rs.next()) {
            model.addRow(new Object[]{
                    rs.getString("code"),
                    rs.getString("name"),
                    rs.getInt("quantity"),
                    rs.getDouble("price"),
                    rs.getString("entryDate")
            });
        }

        if (model.getRowCount() == 0) {
            JOptionPane.showMessageDialog(frame, "هیچ کالایی مطابق با عبارت جستجو پیدا نشد.", "نتیجه جستجو", JOptionPane.INFORMATION_MESSAGE);
        }

    } catch (SQLException e) {
        System.out.println("❌ خطا در جستجوی کالا: " + e.getMessage());
        JOptionPane.showMessageDialog(frame, "خطا در جستجوی کالا در دیتابیس.", "خطا", JOptionPane.ERROR_MESSAGE);
    }
}

private String generateProductCode() {
    return "P" + (int) (Math.random() * 10000);
}

private void displayTime() {
    String currentTime = new SimpleDateFormat("HH:mm:ss").format(new Date());
    String currentDate = new SimpleDateFormat("yyyy-MM-dd").format(new Date());
    JFrame timeFrame = new JFrame("زمان و تاریخ");
    timeFrame.setSize(250, 100);
    timeFrame.setDefaultCloseOperation(JFrame.DISPOSE_ON_CLOSE);

    JLabel timeLabel = new JLabel("زمان: " + currentTime + " تاریخ: " + currentDate, SwingConstants.CENTER);
    timeFrame.add(timeLabel);
    timeFrame.setLocationRelativeTo(frame);
    timeFrame.setVisible(true);
}
}
private void updateStockAndPrice() {
    int row = table.getSelectedRow();
    if (row == -1) {
        JOptionPane.showMessageDialog(frame, "لطفاً یک کالا را انتخاب کنید.", "خطا", JOptionPane.ERROR_MESSAGE);
        return;
    }

    String code = model.getValueAt(row, 0).toString();
    String newQuantityText = JOptionPane.showInputDialog(frame, "مقدار جدید موجودی را وارد کنید:");
    String newPriceText = JOptionPane.showInputDialog(frame, "قیمت جدید را وارد کنید:");

    if (newQuantityText != null && newPriceText != null) {
        try {
            int newQuantity = Integer.parseInt(newQuantityText);
            double newPrice = Double.parseDouble(newPriceText);

            String sql = "UPDATE products SET quantity = ?, price = ? WHERE code = ?";
            try (Connection conn = DriverManager.getConnection(DB_URL);
                 PreparedStatement pstmt = conn.prepareStatement(sql)) {
                pstmt.setInt(1, newQuantity);
                pstmt.setDouble(2, newPrice);
                pstmt.setString(3, code);
                pstmt.executeUpdate();

                model.setValueAt(newQuantity, row, 2);
                model.setValueAt(newPrice, row, 3);

                JOptionPane.showMessageDialog(frame, "موجودی و قیمت با موفقیت به روزرسانی شد.", "موفقیت", JOptionPane.INFORMATION_MESSAGE);
            }
        } catch (NumberFormatException e) {
            JOptionPane.showMessageDialog(frame, "لطفاً مقدار عددی وارد کنید.", "خطا", JOptionPane.ERROR_MESSAGE);
        } catch (SQLException e) {
            System.out.println("❌ خطا در به روز رسانی کالا: " + e.getMessage());
            JOptionPane.showMessageDialog(frame, "خطا در به روز رسانی اطلاعات کالا.", "خطا", JOptionPane.ERROR_MESSAGE);
        }
    }
}

private void autoSave() {
    // اگر میخواهید سیستم به صورت خودکار هر 5 دقیقه ذخیره کند
    Timer timer = new Timer(300000, new ActionListener() {
        @Override
        public void actionPerformed(ActionEvent e) {
            saveData();
        }
    });
    timer.start();
}

private void saveData() {
    String sql = "SELECT * FROM products";
    try (Connection conn = DriverManager.getConnection(DB_URL);
         Statement stmt = conn.createStatement();
         ResultSet rs = stmt.executeQuery(sql)) {

        FileWriter writer = new FileWriter("warehouse_backup.csv");
        BufferedWriter bufferedWriter = new BufferedWriter(writer);
        bufferedWriter.write("Code, Name, Quantity, Price, EntryDate\n");

        while (rs.next()) {
            bufferedWriter.write(rs.getString("code") + ", "
                    + rs.getString("name") + ", "
                    + rs.getInt("quantity") + ", "
                    + rs.getDouble("price") + ", "
                    + rs.getString("entryDate") + "\n");
        }

        bufferedWriter.close();
        JOptionPane.showMessageDialog(frame, "اطلاعات به صورت خودکار ذخیره شد.", "موفقیت", JOptionPane.INFORMATION_MESSAGE);

    } catch (SQLException | IOException e) {
        System.out.println("❌ خطا در ذخیره داده‌ها: " + e.getMessage());
        JOptionPane.showMessageDialog(frame, "خطا در ذخیره داده‌ها.", "خطا", JOptionPane.ERROR_MESSAGE);
    }
}

private void loadProducts() {
    String sql = "SELECT * FROM products";
    try (Connection conn = DriverManager.getConnection(DB_URL);
         Statement stmt = conn.createStatement();
         ResultSet rs = stmt.executeQuery(sql)) {

        model.setRowCount(0); // Clear existing rows

        while (rs.next()) {
            model.addRow(new Object[]{
                    rs.getString("code"),
                    rs.getString("name"),
                    rs.getInt("quantity"),
                    rs.getDouble("price"),
                    rs.getString("entryDate")
            });
        }

    } catch (SQLException e) {
        System.out.println("❌ خطا در بارگیری کالاها: " + e.getMessage());
        JOptionPane.showMessageDialog(frame, "خطا در بارگیری کالاها.", "خطا", JOptionPane.ERROR_MESSAGE);
    }
}

private void createGUI() {
    frame = new JFrame("سیستم انبارداری");
    frame.setSize(900, 500);
    frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
    frame.setLocationRelativeTo(null);

    JPanel panel = new JPanel();
    panel.setLayout(new BorderLayout());

    // Table setup
    model = new DefaultTableModel(new Object[]{"کد", "نام کالا", "مقدار", "قیمت", "تاریخ ورود"}, 0);
    table = new JTable(model);
    JScrollPane scrollPane = new JScrollPane(table);
    panel.add(scrollPane, BorderLayout.CENTER);

    // Input fields and buttons
    JPanel inputPanel = new JPanel(new GridLayout(5, 2));
    JLabel itemNameLabel = new JLabel("نام کالا:");
    JLabel quantityLabel = new JLabel("مقدار:");
    JLabel priceLabel = new JLabel("قیمت:");

    itemNameField = new JTextField();
    quantityField = new JTextField();
    priceField = new JTextField();

    inputPanel.add(itemNameLabel);
    inputPanel.add(itemNameField);
    inputPanel.add(quantityLabel);
    inputPanel.add(quantityField);
    inputPanel.add(priceLabel);
    inputPanel.add(priceField);

    JButton addButton = new JButton("اضافه کردن کالا");
    addButton.addActionListener(e -> addProduct());
    inputPanel.add(addButton);

    JButton deleteButton = new JButton("حذف کالا");
    deleteButton.addActionListener(e -> deleteProduct());
    inputPanel.add(deleteButton);

    JButton updateButton = new JButton("به روز رسانی کالا");
    updateButton.addActionListener(e -> updateStockAndPrice());
    inputPanel.add(updateButton);

    JButton searchButton = new JButton("جستجو");
    searchButton.addActionListener(e -> searchProduct());
    inputPanel.add(searchButton);

    JButton saveButton = new JButton("ذخیره خودکار");
    saveButton.addActionListener(e -> autoSave());
    inputPanel.add(saveButton);

    panel.add(inputPanel, BorderLayout.NORTH);

    // Time display
    JButton timeButton = new JButton("نمایش تاریخ و زمان");
    timeButton.addActionListener(e -> displayTime());
    panel.add(timeButton, BorderLayout.SOUTH);

    frame.add(panel);
    frame.setVisible(true);
}
}
