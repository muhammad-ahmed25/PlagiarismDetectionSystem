import java.awt.*;
import java.awt.event.*;
import java.io.*;
import java.util.*;
import javax.swing.*;
import javax.swing.filechooser.FileNameExtensionFilter;
import javax.swing.table.DefaultTableModel;
import javax.swing.table.DefaultTableCellRenderer;

public class PlagiarismDetectorGUI extends JFrame {
    private static final long serialVersionUID = 1L;
    private JTextArea resultArea;
    private JTextArea file1Preview;
    private JTextArea file2Preview;
    private JTextField thresholdField;
    private JProgressBar progressBar;
    private JFileChooser fileChooser;
    private File file1;
    private File file2;
    private transient HashMap<String, HashMap<String, Integer>> wordFreqCache;
    private transient HashMap<String, HashMap<String, Integer>> ngramFreqCache;
    private transient HashSet<String> stopWords;
    private static final long MAX_FILE_SIZE = 1_000_000; // 1 MB
    private static final int MAX_WORD_COUNT = 10_000; // 10,000 words
    private static final int MIN_WORD_FREQ = 2; // Minimum word frequency
    private static final int NGRAM_SIZE = 2; // Bigrams
    private static final String ERROR_TITLE = "Error";

    // Result class for storing detection results
    private static class Result implements Serializable {
        private static final long serialVersionUID = 1L;
        String file1Name;
        String file2Name;
        double wordSimilarity;
        double ngramSimilarity;
        boolean isPlagiarized;

        Result(String file1Name, String file2Name, double wordSimilarity, double ngramSimilarity, boolean isPlagiarized) {
            this.file1Name = file1Name;
            this.file2Name = file2Name;
            this.wordSimilarity = wordSimilarity;
            this.ngramSimilarity = ngramSimilarity;
            this.isPlagiarized = isPlagiarized;
        }
    }

    // Manager for handling results
    private static class ResultManager {
        private ArrayList<Result> results;

        ResultManager() {
            results = new ArrayList<>();
        }

        void addResult(String file1Name, String file2Name, double wordSimilarity, double ngramSimilarity, boolean isPlagiarized) {
            results.add(new Result(file1Name, file2Name, wordSimilarity, ngramSimilarity, isPlagiarized));
        }

        void clearResults() {
            results.clear();
        }

        void displayResults(JFrame parent) {
            if (results.isEmpty()) {
                JOptionPane.showMessageDialog(parent, 
                    "No detection results available. Please run plagiarism detection first.", 
                    ERROR_TITLE, JOptionPane.ERROR_MESSAGE);
                return;
            }

            JFrame resultsFrame = new JFrame("Plagiarism Detection Results");
            resultsFrame.setSize(700, 200);
            resultsFrame.setLocationRelativeTo(parent);
            resultsFrame.setDefaultCloseOperation(WindowConstants.DISPOSE_ON_CLOSE);

            String[] columns = {"File 1", "File 2", "Word Similarity (%)", "N-gram Similarity (%)", "Plagiarism Detected"};
            DefaultTableModel model = new DefaultTableModel(columns, 0);
            for (Result result : results) {
                model.addRow(new Object[]{
                    result.file1Name,
                    result.file2Name,
                    String.format("%.2f", result.wordSimilarity),
                    String.format("%.2f", result.ngramSimilarity),
                    result.isPlagiarized ? "Yes" : "No"
                });
            }

            JTable table = new JTable(model);
            table.setDefaultRenderer(Object.class, new DefaultTableCellRenderer() {
                @Override
                public Component getTableCellRendererComponent(JTable table, Object value, boolean isSelected, 
                                                              boolean hasFocus, int row, int column) {
                    Component c = super.getTableCellRendererComponent(table, value, isSelected, hasFocus, row, column);
                    String plagiarized = (String) table.getValueAt(row, 4);
                    c.setBackground(plagiarized.equals("Yes") ? new Color(255, 204, 204) : Color.WHITE);
                    return c;
                }
            });
            JScrollPane scrollPane = new JScrollPane(table);
            resultsFrame.add(scrollPane, BorderLayout.CENTER);
            resultsFrame.setVisible(true);
        }

        void exportResults(JFrame parent) {
            if (results.isEmpty()) {
                JOptionPane.showMessageDialog(parent, 
                    "No results to export. Please run plagiarism detection first.", 
                    ERROR_TITLE, JOptionPane.ERROR_MESSAGE);
                return;
            }

            JFileChooser exportChooser = new JFileChooser();
            exportChooser.setCurrentDirectory(new File(System.getProperty("user.dir")));
            exportChooser.setFileFilter(new FileNameExtensionFilter("Text Files (*.txt)", "txt"));
            if (exportChooser.showSaveDialog(parent) == JFileChooser.APPROVE_OPTION) {
                File file = exportChooser.getSelectedFile();
                if (!file.getName().endsWith(".txt")) {
                    file = new File(file.getAbsolutePath() + ".txt");
                }
                try (PrintWriter writer = new PrintWriter(file)) {
                    writer.println("Plagiarism Detection Results:");
                    for (Result result : results) {
                        writer.printf("File 1: %s%n", result.file1Name);
                        writer.printf("File 2: %s%n", result.file2Name);
                        writer.printf("Word Similarity: %.2f%%%n", result.wordSimilarity);
                        writer.printf("N-gram Similarity: %.2f%%%n", result.ngramSimilarity);
                        writer.printf("Plagiarism Detected: %s%n", result.isPlagiarized ? "Yes" : "No");
                        writer.println("---");
                    }
                    JOptionPane.showMessageDialog(parent, 
                        "Results exported to " + file.getName(), 
                        "Success", JOptionPane.INFORMATION_MESSAGE);
                } catch (IOException e) {
                    JOptionPane.showMessageDialog(parent, 
                        "Error exporting results: " + e.getMessage(), 
                        ERROR_TITLE, JOptionPane.ERROR_MESSAGE);
                }
            }
        }
    }

    // Custom component for similarity gauge
    private static class SimilarityGauge extends JComponent {
        private double similarity;

        SimilarityGauge() {
            setPreferredSize(new Dimension(150, 150));
            this.similarity = 0.0;
        }

        void setSimilarity(double similarity) {
            this.similarity = similarity;
            repaint();
        }

        @Override
        protected void paintComponent(Graphics g) {
            super.paintComponent(g);
            Graphics2D g2d = (Graphics2D) g;
            g2d.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);

            int width = getWidth();
            int height = getHeight();
            int size = Math.min(width, height) - 20;
            int x = (width - size) / 2;
            int y = (height - size) / 2;

            // Background arc
            g2d.setColor(Color.LIGHT_GRAY);
            g2d.fillArc(x, y, size, size, 0, 180);

            // Similarity arc
            int angle = (int) (180 * similarity);
            g2d.setColor(similarity >= 0.5 ? Color.RED : Color.GREEN);
            g2d.fillArc(x, y, size, size, 0, angle);

            // Text
            g2d.setColor(Color.BLACK);
            g2d.setFont(new Font("Arial", Font.BOLD, 14));
            String text = String.format("%.2f%%", similarity * 100);
            FontMetrics fm = g2d.getFontMetrics();
            int textX = x + (size - fm.stringWidth(text)) / 2;
            int textY = y + size / 2 + fm.getAscent() / 2;
            g2d.drawString(text, textX, textY);
        }
    }

    private transient ResultManager resultManager;
    private transient SimilarityGauge similarityGauge;

    public PlagiarismDetectorGUI() {
        try {
            UIManager.setLookAndFeel("javax.swing.plaf.nimbus.NimbusLookAndFeel");
        } catch (Exception e) {
            // Fallback to default Look and Feel
        }

        setTitle("Plagiarism Detection System");
        setDefaultCloseOperation(EXIT_ON_CLOSE);
        setSize(800, 600);
        setLocationRelativeTo(null);

        file1 = null;
        file2 = null;
        wordFreqCache = new HashMap<>();
        ngramFreqCache = new HashMap<>();
        stopWords = initializeStopWords();
        resultManager = new ResultManager();
        similarityGauge = new SimilarityGauge();

        fileChooser = new JFileChooser();
        fileChooser.setCurrentDirectory(new File(System.getProperty("user.dir")));
        fileChooser.setFileSelectionMode(JFileChooser.FILES_ONLY);
        fileChooser.setFileFilter(new FileNameExtensionFilter("Text Files (*.txt)", "txt"));

        JPanel mainPanel = new JPanel(new BorderLayout(10, 10));
        mainPanel.setBorder(BorderFactory.createEmptyBorder(10, 10, 10, 10));

        // Control panel
        JPanel controlPanel = new JPanel(new FlowLayout(FlowLayout.LEFT, 10, 5));
        JButton selectFile1Button = new JButton("Select File 1", createIcon("/select_file.png"));
        selectFile1Button.addActionListener(e -> selectFile1());
        controlPanel.add(selectFile1Button);

        JButton selectFile2Button = new JButton("Select File 2", createIcon("/select_file.png"));
        selectFile2Button.addActionListener(e -> selectFile2());
        controlPanel.add(selectFile2Button);

        JLabel thresholdLabel = new JLabel("Threshold (0-1):");
        controlPanel.add(thresholdLabel);
        thresholdField = new JTextField("0.5", 5);
        controlPanel.add(thresholdField);

        JButton detectButton = new JButton("Detect Plagiarism", createIcon("/detect.png"));
        detectButton.addActionListener(e -> detectPlagiarism());
        controlPanel.add(detectButton);

        JButton viewResultsButton = new JButton("View Results", createIcon("/view.png"));
        viewResultsButton.addActionListener(e -> resultManager.displayResults(this));
        controlPanel.add(viewResultsButton);

        JButton exportButton = new JButton("Export Results", createIcon("/export.png"));
        exportButton.addActionListener(e -> resultManager.exportResults(this));
        controlPanel.add(exportButton);

        mainPanel.add(controlPanel, BorderLayout.NORTH);

        // File preview and result panel
        JPanel centerPanel = new JPanel(new GridLayout(1, 3, 10, 10));

        file1Preview = new JTextArea();
        file1Preview.setEditable(false);
        file1Preview.setFont(new Font("Monospaced", Font.PLAIN, 12));
        file1Preview.setBorder(BorderFactory.createTitledBorder("File 1 Preview"));
        centerPanel.add(new JScrollPane(file1Preview));

        file2Preview = new JTextArea();
        file2Preview.setEditable(false);
        file2Preview.setFont(new Font("Monospaced", Font.PLAIN, 12));
        file2Preview.setBorder(BorderFactory.createTitledBorder("File 2 Preview"));
        centerPanel.add(new JScrollPane(file2Preview));

        JPanel resultPanel = new JPanel(new BorderLayout(5, 5));
        resultPanel.setBorder(BorderFactory.createTitledBorder("Results"));
        resultArea = new JTextArea();
        resultArea.setEditable(false);
        resultArea.setFont(new Font("Monospaced", Font.PLAIN, 12));
        resultPanel.add(new JScrollPane(resultArea), BorderLayout.CENTER);
        resultPanel.add(similarityGauge, BorderLayout.SOUTH);
        centerPanel.add(resultPanel);

        mainPanel.add(centerPanel, BorderLayout.CENTER);

        // Progress bar
        progressBar = new JProgressBar();
        progressBar.setStringPainted(true);
        progressBar.setVisible(false);
        mainPanel.add(progressBar, BorderLayout.SOUTH);

        add(mainPanel);
    }

    private ImageIcon createIcon(String path) {
        // Placeholder for icons (replace with actual resources in production)
        return null; // Comment out or replace with actual ImageIcon loading
        // Example: return new ImageIcon(getClass().getResource(path));
    }

    private HashSet<String> initializeStopWords() {
        HashSet<String> words = new HashSet<>();
        Collections.addAll(words, 
            "the", "a", "an", "and", "or", "but", "in", "on", "at", "to",
            "for", "of", "with", "by");
        return words;
    }

    private void selectFile1() {
        resultArea.setText("");
        int returnVal = fileChooser.showOpenDialog(this);
        if (returnVal == JFileChooser.APPROVE_OPTION) {
            File selectedFile = fileChooser.getSelectedFile();
            if (validateFile(selectedFile, file2)) {
                file1 = selectedFile;
                wordFreqCache.remove(file1.getAbsolutePath());
                ngramFreqCache.remove(file1.getAbsolutePath());
                resultArea.append("File 1 selected: " + file1.getName() + "%n");
                if (file2 != null) {
                    resultArea.append("File 2 selected: " + file2.getName() + "%n");
                }
                updateFilePreview(file1, file1Preview);
            } else {
                resultArea.append("Invalid selection for File 1. Must be a .txt file, ≤ 1MB, and different from File 2.%n");
            }
        } else {
            resultArea.append("File 1 selection cancelled.%n");
        }
    }

    private void selectFile2() {
        resultArea.setText("");
        int returnVal = fileChooser.showOpenDialog(this);
        if (returnVal == JFileChooser.APPROVE_OPTION) {
            File selectedFile = fileChooser.getSelectedFile();
            if (validateFile(selectedFile, file1)) {
                file2 = selectedFile;
                wordFreqCache.remove(file2.getAbsolutePath());
                ngramFreqCache.remove(file2.getAbsolutePath());
                resultArea.append("File 2 selected: " + file2.getName() + "%n");
                if (file1 != null) {
                    resultArea.append("File 1 selected: " + file1.getName() + "%n");
                }
                updateFilePreview(file2, file2Preview);
            } else {
                resultArea.append("Invalid selection for File 2. Must be a .txt file, ≤ 1MB, and different from File 1.%n");
            }
        } else {
            resultArea.append("File 2 selection cancelled.%n");
        }
    }

    private boolean validateFile(File file, File otherFile) {
        return file != null &&
               file.isFile() &&
               file.getName().toLowerCase().endsWith(".txt") &&
               file.length() <= MAX_FILE_SIZE &&
               (otherFile == null || !file.getAbsolutePath().equals(otherFile.getAbsolutePath()));
    }

    private void updateFilePreview(File file, JTextArea previewArea) {
        previewArea.setText("");
        try (BufferedReader reader = new BufferedReader(new FileReader(file))) {
            StringBuilder content = new StringBuilder();
            String line;
            int maxLines = 10; // Limit preview to 10 lines
            while ((line = reader.readLine()) != null && maxLines-- > 0) {
                content.append(line).append("\n");
            }
            previewArea.setText(content.toString());
            previewArea.setCaretPosition(0);
        } catch (IOException e) {
            previewArea.setText("Error loading preview: " + e.getMessage());
        }
    }

    private void detectPlagiarism() {
        if (file1 == null || file2 == null) {
            JOptionPane.showMessageDialog(this, 
                "Please select both File 1 and File 2 (.txt files).", 
                ERROR_TITLE, JOptionPane.ERROR_MESSAGE);
            return;
        }

        double threshold;
        try {
            threshold = Double.parseDouble(thresholdField.getText());
            if (threshold < 0 || threshold > 1) {
                JOptionPane.showMessageDialog(this, 
                    "Threshold must be between 0 and 1.", 
                    ERROR_TITLE, JOptionPane.ERROR_MESSAGE);
                return;
            }
        } catch (NumberFormatException e) {
            JOptionPane.showMessageDialog(this, 
                "Invalid threshold. Please enter a number between 0 and 1.", 
                ERROR_TITLE, JOptionPane.ERROR_MESSAGE);
            return;
        }

        progressBar.setValue(0);
        progressBar.setVisible(true);
        resultArea.setText("Detecting plagiarism...%n");

        SwingWorker<Void, Integer> worker = new SwingWorker<>() {
            @Override
            protected Void doInBackground() throws Exception {
                publish(10);
                HashMap<String, Integer> wordFreq1 = getWordFrequencies(file1.getAbsolutePath());
                publish(30);
                HashMap<String, Integer> wordFreq2 = getWordFrequencies(file2.getAbsolutePath());
                publish(50);

                if (wordFreq1.isEmpty() || wordFreq2.isEmpty()) {
                    resultArea.append("Error: One or both files have no valid words after processing.%n");
                    return null;
                }

                HashMap<String, Integer> ngramFreq1 = getNgramFrequencies(file1.getAbsolutePath());
                publish(70);
                HashMap<String, Integer> ngramFreq2 = getNgramFrequencies(file2.getAbsolutePath());
                publish(90);

                double wordSimilarity = calculateFrequencySimilarity(wordFreq1, wordFreq2);
                double ngramSimilarity = calculateFrequencySimilarity(ngramFreq1, ngramFreq2);
                boolean isPlagiarized = wordSimilarity >= threshold || ngramSimilarity >= threshold;

                resultManager.clearResults();
                resultManager.addResult(file1.getName(), file2.getName(), wordSimilarity * 100, ngramSimilarity * 100, isPlagiarized);

                StringBuilder results = new StringBuilder();
                results.append(String.format("Word Similarity: %.2f%%%n", wordSimilarity * 100));
                results.append(String.format("N-gram Similarity: %.2f%%%n", ngramSimilarity * 100));
                results.append(String.format("Word count (File 1): %d%n", wordFreq1.size()));
                results.append(String.format("Word count (File 2): %d%n", wordFreq2.size()));
                results.append(String.format("N-gram count (File 1): %d%n", ngramFreq1.size()));
                results.append(String.format("N-gram count (File 2): %d%n", ngramFreq2.size()));
                if (isPlagiarized) {
                    results.append("*** Potential plagiarism detected! ***%n");
                }

                resultArea.append(results.toString());
                similarityGauge.setSimilarity(wordSimilarity);
                publish(100);
                return null;
            }

            @Override
            protected void process(java.util.List<Integer> chunks) {
                progressBar.setValue(chunks.get(chunks.size() - 1));
            }

            @Override
            protected void done() {
                progressBar.setVisible(false);
            }
        };
        worker.execute();
    }

    private String readFile(String filePath) throws IOException {
        StringBuilder content = new StringBuilder();
        try (BufferedReader reader = new BufferedReader(new FileReader(filePath))) {
            String line;
            while ((line = reader.readLine()) != null) {
                content.append(line).append(" ");
            }
        }
        return content.toString();
    }

    private String stemWord(String word) {
        if (word.length() <= 2) {
            return word;
        }
        if (word.endsWith("ing") && word.length() > 5) {
            return word.substring(0, word.length() - 3);
        }
        if (word.endsWith("ed") && word.length() > 4) {
            return word.substring(0, word.length() - 2);
        }
        if (word.endsWith("es") && word.length() > 4) {
            return word.substring(0, word.length() - 2);
        }
        if (word.endsWith("s") && word.length() > 3) {
            return word.substring(0, word.length() - 1);
        }
        if (word.endsWith("ly") && word.length() > 4) {
            return word.substring(0, word.length() - 2);
        }
        return word;
    }

    private HashMap<String, Integer> getWordFrequencies(String filePath) throws IOException {
        if (wordFreqCache.containsKey(filePath)) {
            return new HashMap<>(wordFreqCache.get(filePath));
        }

        String text = readFile(filePath);
        String cleaned = text.toLowerCase().replaceAll("[^a-zA-Z0-9\\s]", "");
        ArrayList<String> words = new ArrayList<>();
        for (String word : cleaned.split("\\s+")) {
            if (!word.isEmpty() && !stopWords.contains(word)) {
                words.add(stemWord(word));
            }
        }

        if (words.isEmpty()) {
            return new HashMap<>();
        }

        if (words.size() > MAX_WORD_COUNT) {
            words = new ArrayList<>(words.subList(0, MAX_WORD_COUNT));
        }

        HashMap<String, Integer> wordFreq = new HashMap<>();
        for (String word : words) {
            wordFreq.compute(word, (k, v) -> (v == null) ? 1 : v + 1);
        }

        wordFreq.entrySet().removeIf(entry -> entry.getValue() < MIN_WORD_FREQ);

        wordFreqCache.put(filePath, new HashMap<>(wordFreq));
        return wordFreq;
    }

    private HashMap<String, Integer> getNgramFrequencies(String filePath) throws IOException {
        if (ngramFreqCache.containsKey(filePath)) {
            return new HashMap<>(ngramFreqCache.get(filePath));
        }

        String text = readFile(filePath);
        String cleaned = text.toLowerCase().replaceAll("[^a-zA-Z0-9\\s]", "");
        String[] words = cleaned.split("\\s+");
        ArrayList<String> ngrams = new ArrayList<>();

        for (int i = 0; i < words.length - NGRAM_SIZE + 1; i++) {
            if (words[i].isEmpty() || stopWords.contains(words[i])) {
                continue;
            }
            StringBuilder ngram = new StringBuilder(stemWord(words[i]));
            boolean valid = true;
            for (int j = 1; j < NGRAM_SIZE; j++) {
                if (i + j >= words.length || words[i + j].isEmpty() || stopWords.contains(words[i + j])) {
                    valid = false;
                    break;
                }
                ngram.append(" ").append(stemWord(words[i + j]));
            }
            if (valid) {
                ngrams.add(ngram.toString());
            }
        }

        if (ngrams.isEmpty()) {
            return new HashMap<>();
        }

        HashMap<String, Integer> ngramFreq = new HashMap<>();
        for (String ngram : ngrams) {
            ngramFreq.compute(ngram, (k, v) -> (v == null) ? 1 : v + 1);
        }

        ngramFreq.entrySet().removeIf(entry -> entry.getValue() < MIN_WORD_FREQ);

        ngramFreqCache.put(filePath, new HashMap<>(ngramFreq));
        return ngramFreq;
    }

    private double calculateFrequencySimilarity(HashMap<String, Integer> freq1, HashMap<String, Integer> freq2) {
        double dotProduct = 0.0;
        for (Map.Entry<String, Integer> entry : freq1.entrySet()) {
            String key = entry.getKey();
            if (freq2.containsKey(key)) {
                dotProduct += entry.getValue() * freq2.get(key);
            }
        }

        double norm1 = 0.0;
        for (int count : freq1.values()) {
            norm1 += count * count;
        }
        norm1 = Math.sqrt(norm1);

        double norm2 = 0.0;
        for (int count : freq2.values()) {
            norm2 += count * count;
        }
        norm2 = Math.sqrt(norm2);

        return (norm1 * norm2 == 0) ? 0.0 : dotProduct / (norm1 * norm2);
    }

    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> new PlagiarismDetectorGUI().setVisible(true));
    }
}
