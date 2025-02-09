# Proje-3
Authors and Collaborations
package proje3;

import javafx.application.Application;
import javafx.application.Platform;
import javafx.concurrent.Task;
import javafx.scene.Scene;
import javafx.scene.canvas.Canvas;
import javafx.scene.canvas.GraphicsContext;
import javafx.scene.control.Button;
import javafx.scene.control.TextArea;
import javafx.scene.control.TextInputDialog;
import javafx.scene.input.MouseEvent;
import javafx.scene.input.ScrollEvent;
import javafx.scene.layout.BorderPane;
import javafx.scene.layout.VBox;
import javafx.stage.Stage;

import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;
import java.util.*;





public class proje3 extends Application {

    private Canvas graphCanvas;
    private TextArea processLog;
    private Map<String, Map<String, Integer>> graph;
    private Map<String, String> authorMap;
    private double translateX = 0;
    private double translateY = 0;
    private double scale = 1.0;
    
    private double startX = 0;
    private double startY = 0;

    
    private final Map<String, double[]> nodePositions = new HashMap<>();

    public static void main(String[] args) {
        launch(args);
    }

    @Override
    public void start(Stage primaryStage) {
        graph = new HashMap<>();
        authorMap = new HashMap<>();

        BorderPane root = new BorderPane();

        
        
        processLog = new TextArea();
        processLog.setEditable(false);
        processLog.setPrefWidth(250);
        root.setLeft(processLog);

        
        
        VBox buttonPanel = new VBox(10);
        buttonPanel.setPrefWidth(150);
        addButtons(buttonPanel); 
        root.setRight(buttonPanel);

        
        graphCanvas = new Canvas(600, 600);
        graphCanvas.setOnScroll(this::handleZoom);
        root.setCenter(graphCanvas);

       
        loadGraphFromCSV("C:\\Users\\senag\\OneDrive\\Masaüstü\\PROLAB 3 - GÜNCEL DATASET(in).csv");

        
        for (String node : graph.keySet()) {
            nodePositions.putIfAbsent(node, new double[]{Math.random() * 400, Math.random() * 400});
        }

        applyForceDirectedLayout(); 
        applyCircularLayout();      

        
        graphCanvas.setOnMousePressed(event -> {
            startX = event.getX();
            startY = event.getY();
        });

        graphCanvas.setOnMouseDragged(event -> {
            translateX += event.getX() - startX;
            translateY += event.getY() - startY;
            startX = event.getX();
            startY = event.getY();
            drawGraph(); 
        });

        
        Scene scene = new Scene(root, 800, 600);
        primaryStage.setTitle("Grafik Görselleştirme Uygulaması");
        primaryStage.setScene(scene);
        primaryStage.show();

        log("Veriler yüklendi. Lütfen butonları kullanarak graf işlemlerini gerçekleştirin.");
    }

    
    
    
    


    private void addButtons(VBox buttonPanel) {
        Button handleShortestPath = new Button("1. İster: En Kısa Yol");
        handleShortestPath.setOnAction(e -> simulateShortestPath());

        Button handleCoauthorQueue = new Button("2. İster: Kuyruk Oluşturma");
        handleCoauthorQueue.setOnAction(e -> calculateCoauthorQueue());

        Button handleBSTCreation = new Button("3. İster: BST Oluşturma");
        handleBSTCreation.setOnAction(e -> createAndModifyBST());

        Button handleAllShortestPaths = new Button("4. İster: Tüm Kısa Yollar");
        handleAllShortestPaths.setOnAction(e -> calculateShortestPaths());

        Button handleCoauthorCount = new Button("5. İster: İşbirlikçi Sayısı");
        handleCoauthorCount.setOnAction(e -> calculateCoauthorCount());

        Button handleMostCollaborative = new Button("6. İster: En İşbirlikçi Yazar");
        handleMostCollaborative.setOnAction(e -> findMostCollaborativeAuthor());

        Button handleLongestPath = new Button("7. İster: En Uzun Yol");
        handleLongestPath.setOnAction(e -> findLongestPath());

        // Yeni butonu tanımla
        Button highlightAuthorGraphButton = new Button("Yazar Ağını Vurgula");
        highlightAuthorGraphButton.setOnAction(e -> highlightGraphForAuthor());

        // Tüm butonları ekle
        buttonPanel.getChildren().addAll(
                handleShortestPath,
                handleCoauthorQueue,
                handleBSTCreation,
                handleAllShortestPaths,
                handleCoauthorCount,
                handleMostCollaborative,
                handleLongestPath,
                highlightAuthorGraphButton
        );
    }
    
    
    
    
    
    
    
    
    private void simulateShortestPath() {
        TextInputDialog dialog = new TextInputDialog();
        dialog.setTitle("En Kısa Yol");
        dialog.setHeaderText("A ve B yazarlarının ID'lerini virgül ile ayırarak girin:");
        String input = dialog.showAndWait().orElse(null);

        if (input == null || !input.contains(",")) {
            log("Hatalı giriş yaptınız. Lütfen iki ID'yi virgül ile ayırarak giriniz.");
            return;
        }

        String[] authors = input.split(",");
        String author1 = authors[0].trim();
        String author2 = authors[1].trim();

        if (!graph.containsKey(author1) || !graph.containsKey(author2)) {
            log("Girilen yazarlardan biri veya her ikisi grafikte mevcut değil.");
            return;
        }

       
        Task<Void> shortestPathTask = new Task<>() {
            @Override
            protected Void call() throws Exception {
                
                Map<String, Integer> distances = new HashMap<>();
                Map<String, String> previous = new HashMap<>();
                PriorityQueue<String> queue = new PriorityQueue<>(Comparator.comparingInt(distances::get));

                for (String node : graph.keySet()) {
                    distances.put(node, Integer.MAX_VALUE); 
                }
                distances.put(author1, 0); 
                queue.add(author1);

                while (!queue.isEmpty()) {
                    String current = queue.poll();

                    
                    updateMessage("Kuyruk: " + queue);

                    for (Map.Entry<String, Integer> neighbor : graph.get(current).entrySet()) {
                        String neighborNode = neighbor.getKey();
                        int weight = neighbor.getValue();

                        int newDist = distances.get(current) + weight;
                        if (newDist < distances.get(neighborNode)) {
                            distances.put(neighborNode, newDist);
                            previous.put(neighborNode, current);
                            queue.add(neighborNode);
                        }
                    }
                }

                
                if (!previous.containsKey(author2)) {
                    updateMessage("A ve B arasında bir yol bulunamadı.");
                    return null;
                }

                
                List<String> path = new ArrayList<>();
                for (String at = author2; at != null; at = previous.get(at)) {
                    path.add(at);
                }
                Collections.reverse(path);

               
                updateMessage("En kısa yol: " + path);
                updateMessage("A'dan B'ye toplam ağırlık: " + distances.get(author2));

                
                Platform.runLater(() -> drawShortestPath(path));
                return null;
            }
        };

     
        shortestPathTask.messageProperty().addListener((obs, oldMessage, newMessage) -> log(newMessage));

        
        new Thread(shortestPathTask).start();
    }

    
    
    
    
    
    
    
    
    
    
    private void debugGraph() {
        log("Grafik Hata Ayıklama - Düğümler ve Bağlantılar:");
        for (String node : graph.keySet()) {
            log("Node: " + node + ", Connections: " + graph.get(node));
        }

        log("AuthorMap Debug - Authors:");
        for (String id : authorMap.keySet()) {
            log("Author ID: " + id + ", Name: " + authorMap.get(id));
        }
    }
    
    
    
    
    
    

    private void calculateCoauthorQueue() {
        
        TextInputDialog dialog = new TextInputDialog();
        dialog.setTitle("Ortak Yazar Kuyruğu");
        dialog.setHeaderText("Yazar kimliğini girin:");
        String authorId = dialog.showAndWait().orElse(null);

        if (authorId == null || !graph.containsKey(authorId)) {
            log("Geçersiz veya mevcut olmayan yazar kimliği.");
            return;
        }

      
        Map<String, Integer> coauthors = graph.get(authorId);
        if (coauthors == null || coauthors.isEmpty()) {
            log("Bu yazar için ortak yazar bulunamadı.");
            return;
        }

       
        PriorityQueue<Map.Entry<String, Integer>> queue = new PriorityQueue<>(
                (a, b) -> b.getValue().compareTo(a.getValue())
        );
        queue.addAll(coauthors.entrySet());

        log("Yazar için ortak yazar kuyruğu: " + authorMap.getOrDefault(authorId, authorId));

        
        Set<String> highlightedNodes = new HashSet<>();
        highlightedNodes.add(authorId); 

        while (!queue.isEmpty()) {
            Map.Entry<String, Integer> coauthor = queue.poll();
            log("Coauthor: " + authorMap.getOrDefault(coauthor.getKey(), coauthor.getKey())
                    + ", Collaboration count: " + coauthor.getValue());
            highlightedNodes.add(coauthor.getKey());
        }

        
        highlightAuthorAndCoauthors(authorId, highlightedNodes);
    }
    
    
    
    
    
    private void highlightAuthorAndCoauthors(String authorId, Set<String> highlightedNodes) {
        GraphicsContext gc = graphCanvas.getGraphicsContext2D();
        gc.clearRect(0, 0, graphCanvas.getWidth(), graphCanvas.getHeight());

        
        for (String src : highlightedNodes) {
            for (String dest : graph.getOrDefault(src, Collections.emptyMap()).keySet()) {
                if (highlightedNodes.contains(dest)) {
                    double[] srcPos = nodePositions.get(src);
                    double[] destPos = nodePositions.get(dest);

                    if (srcPos != null && destPos != null) {
                        gc.setStroke(javafx.scene.paint.Color.LIGHTGRAY);
                        gc.setLineWidth(1.5); // İnce gri çizgi
                        gc.strokeLine(srcPos[0], srcPos[1], destPos[0], destPos[1]);
                    }
                }
            }
        }

       
        for (String node : highlightedNodes) {
            double[] pos = nodePositions.get(node);
            if (pos != null) {
                int collaborationCount = graph.get(node).size();
                double radius = node.equals(authorId) ? 20 : 12; 
                
                
                gc.setFill(node.equals(authorId) ? javafx.scene.paint.Color.MAGENTA : javafx.scene.paint.Color.CADETBLUE);
                gc.fillOval(pos[0] - radius, pos[1] - radius, radius * 2, radius * 2);

                
                gc.setStroke(javafx.scene.paint.Color.BLACK);
                gc.strokeOval(pos[0] - radius, pos[1] - radius, radius * 2, radius * 2);

                
                gc.setFill(javafx.scene.paint.Color.BLACK);
                gc.fillText(authorMap.getOrDefault(node, node), pos[0] + 5, pos[1] - 5);
            }
        }
    }

    
    
    
    
    
    private void calculateShortestPathsForAuthor() {
   
        
        TextInputDialog dialog = new TextInputDialog();
        dialog.setTitle("Kısa Yolların Hesaplanması");
        dialog.setHeaderText("Lütfen bir yazar ID'si girin:");
        String authorId = dialog.showAndWait().orElse(null);

        if (authorId == null || !graph.containsKey(authorId)) {
            log("Geçersiz veya grafikte bulunmayan bir yazar ID'si girdiniz.");
            return;
        }

        
        log("En kısa yollar hesaplanıyor...");
        Map<String, Integer> distances = dijkstraShortestPaths(authorId);
        log("Hesaplama tamamlandı. Sonuçlar:");

      
        for (String target : distances.keySet()) {
            int distance = distances.get(target);
            log(authorId + " -> " + target + " = " + (distance == Integer.MAX_VALUE ? "∞" : distance));
        }
    }

    
    
    
    
    private Map<String, Integer> dijkstraShortestPaths(String startNode) {
    	
        Map<String, Integer> distances = new HashMap<>();
        PriorityQueue<String> queue = new PriorityQueue<>(Comparator.comparingInt(distances::get));
        Set<String> visited = new HashSet<>();

       
        for (String node : graph.keySet()) {
            distances.put(node, Integer.MAX_VALUE);
        }
        distances.put(startNode, 0);
        queue.add(startNode);

        while (!queue.isEmpty()) {
            String current = queue.poll();
            if (!visited.add(current)) continue; 

            for (Map.Entry<String, Integer> neighbor : graph.get(current).entrySet()) {
                String neighborNode = neighbor.getKey();
                int weight = neighbor.getValue();
                int newDistance = distances.get(current) + weight;

                if (newDistance < distances.get(neighborNode)) {
                    distances.put(neighborNode, newDistance);
                    queue.add(neighborNode);
                }
            }
        }

        return distances;
    }
    
    
    
    
    

    private void floydWarshallShortestPaths(Map<String, Map<String, Integer>> subGraph, String startNode) {
        
        Map<String, Map<String, Integer>> distances = new HashMap<>();
        for (String node : subGraph.keySet()) {
            distances.put(node, new HashMap<>());
            for (String target : subGraph.keySet()) {
                if (node.equals(target)) {
                    distances.get(node).put(target, 0); 
                } else {
                    distances.get(node).put(target, Integer.MAX_VALUE); 
                }
            }
            for (String neighbor : subGraph.get(node).keySet()) {
                distances.get(node).put(neighbor, subGraph.get(node).get(neighbor));
            }
        }

        log("Başlangıç mesafe tablosu:");
        logDistanceTable(distances);

        
        for (String k : subGraph.keySet()) {
            for (String i : subGraph.keySet()) {
                for (String j : subGraph.keySet()) {
                    int distIK = distances.get(i).get(k);
                    int distKJ = distances.get(k).get(j);

                    
                    if (distIK != Integer.MAX_VALUE && distKJ != Integer.MAX_VALUE) {
                        int newDist = distIK + distKJ;
                        if (newDist < distances.get(i).get(j)) {
                            distances.get(i).put(j, newDist);
                        }
                    }
                }
            }

            
            log("Düğüm '" + k + "' göz önüne alındığında güncellenen mesafe tablosu:");
            logDistanceTable(distances);
        }

        
        log("Sonuç: Başlangıç düğümü '" + startNode + "' için en kısa yollar:");
        for (String target : distances.get(startNode).keySet()) {
            int dist = distances.get(startNode).get(target);
            log(startNode + " -> " + target + " = " + (dist == Integer.MAX_VALUE ? "∞" : dist));
        }
    }
    
    
    

    private void logDistanceTable(Map<String, Map<String, Integer>> distances) {
        StringBuilder sb = new StringBuilder();
        sb.append("     ");
        for (String col : distances.keySet()) {
            sb.append(String.format("%10s", col));
        }
        sb.append("\n");

        for (String row : distances.keySet()) {
            sb.append(String.format("%-5s", row));
            for (String col : distances.keySet()) {
                int value = distances.get(row).get(col);
                sb.append(String.format("%10s", value == Integer.MAX_VALUE ? "∞" : value));
            }
            sb.append("\n");
        }

        log(sb.toString());
    }


    
    
    

    private List<String> bfsShortestPath(String start, String end) {
        Queue<List<String>> queue = new LinkedList<>();
        Set<String> visited = new HashSet<>();

        queue.add(Collections.singletonList(start));
        visited.add(start);

        while (!queue.isEmpty()) {
            List<String> path = queue.poll();
            String lastNode = path.get(path.size() - 1);

            if (lastNode.equals(end)) {
                return path;
            }

            for (String neighbor : graph.getOrDefault(lastNode, Collections.emptyMap()).keySet()) {
                if (!visited.contains(neighbor)) {
                    visited.add(neighbor);
                    List<String> newPath = new ArrayList<>(path);
                    newPath.add(neighbor);
                    queue.add(newPath);
                }
            }
        }
        return Collections.emptyList(); 
    }

    private void drawShortestPath(List<String> path) {
        GraphicsContext gc = graphCanvas.getGraphicsContext2D();
        gc.clearRect(0, 0, graphCanvas.getWidth(), graphCanvas.getHeight()); 
        

        for (String src : graph.keySet()) {
            for (String dest : graph.get(src).keySet()) {
                double[] srcPos = nodePositions.get(src);
                double[] destPos = nodePositions.get(dest);

                if (srcPos != null && destPos != null) {
                    gc.setStroke(javafx.scene.paint.Color.LIGHTGRAY);
                    gc.setLineWidth(1);
                    gc.strokeLine(srcPos[0], srcPos[1], destPos[0], destPos[1]);
                }
            }
        }

        for (int i = 0; i < path.size() - 1; i++) {
            String src = path.get(i);
            String dest = path.get(i + 1);

            double[] srcPos = nodePositions.get(src);
            double[] destPos = nodePositions.get(dest);

            gc.setStroke(javafx.scene.paint.Color.DEEPSKYBLUE);
            gc.setLineWidth(3);
            gc.strokeLine(srcPos[0], srcPos[1], destPos[0], destPos[1]);
        }

        for (String node : path) {
            double[] pos = nodePositions.get(node);
            gc.setFill(javafx.scene.paint.Color.YELLOW);
            gc.fillOval(pos[0] - 10, pos[1] - 10, 20, 20);

            gc.setFill(javafx.scene.paint.Color.BLACK);
            gc.fillText(authorMap.getOrDefault(node, node), pos[0] + 15, pos[1] - 5);
        }
    }








    private void loadGraphFromCSV(String fileName) {
        try (BufferedReader br = new BufferedReader(new FileReader(fileName))) {
            String line;
            boolean isFirstLine = true;
            while ((line = br.readLine()) != null) {
                if (isFirstLine) {
                    isFirstLine = false;
                    continue; 
                }

                String[] parts = line.split(";"); 
                if (parts.length < 6) continue;

                String orcid = parts[0].trim(); 
                String authorName = parts[3].trim(); 
                String coauthors = parts[4].trim().replaceAll("[\\[\\]'\"]", ""); 

                authorMap.put(orcid, authorName.isEmpty() ? "Yazar " + orcid : authorName);

                if (orcid.isEmpty() || coauthors.isEmpty()) continue;

                for (String coauthor : coauthors.split(",")) {
                    coauthor = coauthor.trim();
                    if (!coauthor.isEmpty()) {
                        addEdge(orcid, coauthor);
                    }
                }
            }
            log("Grafik başarıyla yüklendi.");
        } catch (IOException e) {
            e.printStackTrace();
            log("CSV'den grafik yüklenirken hata oluştu: " + e.getMessage());
        }
    }


    
    private void loadGraphFromDataset(String filePath) {
        try (BufferedReader br = new BufferedReader(new FileReader(filePath))) {
            String line;
            boolean isFirstLine = true;
            while ((line = br.readLine()) != null) {
                if (isFirstLine) { 
                    isFirstLine = false;
                    continue;
                }

                String[] parts = line.split("\t");
                if (parts.length < 6) continue;

                String orcid = parts[0].trim();
                String authorName = parts[3].trim();
                String coauthors = parts[4].trim().replaceAll("[\\[\\]'\"]", "");

                authorMap.put(orcid, authorName);

                for (String coauthor : coauthors.split(",")) {
                    coauthor = coauthor.trim();
                    if (!coauthor.isEmpty()) {
                        addEdge(orcid, coauthor);
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
            log("Veri seti yüklenirken hata oluştu: " + e.getMessage());
        }
    }


    private void addEdge(String src, String dest) {
        if (!graph.containsKey(src)) {
            graph.put(src, new HashMap<>());
        }
        if (!graph.containsKey(dest)) {
            graph.put(dest, new HashMap<>());
        }
        graph.get(src).merge(dest, 1, Integer::sum);
        graph.get(dest).merge(src, 1, Integer::sum);
    }


    private void handleNodeClick(MouseEvent event) {
        double x = event.getX();
        double y = event.getY();

        for (Map.Entry<String, double[]> entry : nodePositions.entrySet()) {
            double[] pos = entry.getValue();
            if (Math.hypot(pos[0] - x, pos[1] - y) <= 10) { 
                String authorId = entry.getKey();
                highlightNode(authorId);
                log("Selected Author: " + authorMap.get(authorId) + "\nCollaborators: " + graph.get(authorId).keySet());
                return;
            }
        }
    }
    private void highlightNode(String authorId) {
        GraphicsContext gc = graphCanvas.getGraphicsContext2D();
        double[] pos = nodePositions.get(authorId);
        if (pos != null) {
            gc.setStroke(javafx.scene.paint.Color.MAGENTA);
            gc.setLineWidth(3);
            gc.strokeOval(pos[0] - 15, pos[1] - 15, 30, 30); 
        }
    }
    private void applyCircularLayout() {
        double centerX = graphCanvas.getWidth() / 2;
        double centerY = graphCanvas.getHeight() / 2;
        double radius = Math.min(centerX, centerY) * 0.7; 

        int n = graph.size();
        int i = 0;

        for (String node : graph.keySet()) {
            double angle = 2 * Math.PI * i / n;
            double x = centerX + radius * Math.cos(angle);
            double y = centerY + radius * Math.sin(angle);
            nodePositions.put(node, new double[]{x, y});
            i++;
        }
    }

    private void drawGraph() {
        GraphicsContext gc = graphCanvas.getGraphicsContext2D();
        gc.clearRect(0, 0, graphCanvas.getWidth(), graphCanvas.getHeight()); 

        gc.save(); 

        
        gc.translate(translateX, translateY);
        gc.scale(scale, scale);

        for (String src : graph.keySet()) {
            for (String dest : graph.get(src).keySet()) {
                double[] srcPos = nodePositions.get(src);
                double[] destPos = nodePositions.get(dest);

                if (srcPos != null && destPos != null) {
                    gc.setStroke(javafx.scene.paint.Color.LIGHTGRAY);
                    gc.setLineWidth(1);
                    gc.strokeLine(srcPos[0], srcPos[1], destPos[0], destPos[1]);
                }
            }
        }

        for (String node : graph.keySet()) {
            double[] pos = nodePositions.get(node);
            if (pos != null) {
                int degree = graph.get(node).size();
                double radius = 10 + Math.log(degree + 1) * 5;

                gc.setFill(javafx.scene.paint.Color.CADETBLUE);
                gc.fillOval(pos[0] - radius, pos[1] - radius, radius * 2, radius * 2);

                gc.setStroke(javafx.scene.paint.Color.BLACK);
                gc.strokeOval(pos[0] - radius, pos[1] - radius, radius * 2, radius * 2);

                gc.setFill(javafx.scene.paint.Color.BLACK);
                gc.fillText(node, pos[0] + radius + 5, pos[1]);
            }
        }

        gc.restore(); 
    }

    
    private javafx.scene.paint.Color getEdgeColor(int weight) {
        if (weight > 5) return javafx.scene.paint.Color.DEEPSKYBLUE; 
        if (weight > 2) return javafx.scene.paint.Color.CORNFLOWERBLUE; 
        return javafx.scene.paint.Color.LIGHTBLUE; 
    }

    private javafx.scene.paint.Color getNodeColor(int paperCount, double averagePapers) {
        if (paperCount > averagePapers * 1.2) return javafx.scene.paint.Color.DARKBLUE; 
        if (paperCount < averagePapers * 0.8) return javafx.scene.paint.Color.ALICEBLUE; 
        return javafx.scene.paint.Color.CORNFLOWERBLUE; 
    }




    private double getNodeRadius(int paperCount, double averagePapers) {
        if (paperCount > averagePapers * 1.2) return 15;
        if (paperCount < averagePapers * 0.8) return 8;
        return 10;
    }

    private double getAveragePaperCount() {
        int totalPapers = 0;
        for (String node : graph.keySet()) {
            totalPapers += graph.get(node).size();
        }
        return totalPapers / (double) graph.size();
    }




    private void handleZoom(ScrollEvent event) {
        double zoomFactor = event.getDeltaY() > 0 ? 1.1 : 0.9;
        scale = Math.max(0.5, Math.min(scale * zoomFactor, 5.0));
        translateX += (event.getX() - graphCanvas.getWidth() / 2) * (1 - zoomFactor);
        translateY += (event.getY() - graphCanvas.getHeight() / 2) * (1 - zoomFactor);
        drawGraph();
    }

    private void applyForceDirectedLayout() {
        double area = 400 * 400;
        double k = Math.sqrt(area / graph.size());
        int maxIterations = 100; 

        for (int iter = 0; iter < maxIterations; iter++) {
            Map<String, double[]> displacements = new HashMap<>();

            for (String node : graph.keySet()) {
                displacements.put(node, new double[2]);
            }

            for (String v : graph.keySet()) {
                double[] d = displacements.get(v);
                for (String u : graph.keySet()) {
                    if (!v.equals(u)) {
                        double[] posV = nodePositions.get(v);
                        double[] posU = nodePositions.get(u);

                        double dx = posV[0] - posU[0];
                        double dy = posV[1] - posU[1];
                        double dist = Math.max(Math.sqrt(dx * dx + dy * dy), 0.1);

                        double force = k * k / dist;
                        d[0] += (dx / dist) * force;
                        d[1] += (dy / dist) * force;
                    }
                }
            }

            for (String v : graph.keySet()) {
                for (String u : graph.get(v).keySet()) {
                    double[] posV = nodePositions.get(v);
                    double[] posU = nodePositions.get(u);

                    double dx = posU[0] - posV[0];
                    double dy = posU[1] - posV[1];
                    double dist = Math.max(Math.sqrt(dx * dx + dy * dy), 0.1);

                    double force = (dist * dist) / k;
                    displacements.get(v)[0] += (dx / dist) * force;
                    displacements.get(v)[1] += (dy / dist) * force;
                    displacements.get(u)[0] -= (dx / dist) * force;
                    displacements.get(u)[1] -= (dy / dist) * force;
                }
            }

            for (String node : graph.keySet()) {
                double[] pos = nodePositions.get(node);
                double[] d = displacements.get(node);

                pos[0] += d[0] * 0.1; 
                pos[1] += d[1] * 0.1;
            }
        }
    }





    private void calculateShortestPaths() {
        TextInputDialog dialog = new TextInputDialog();
        dialog.setTitle("Yazar için En Kısa Yollar");
        dialog.setHeaderText("Bir yazar kimliği girin:");
        String authorId = dialog.showAndWait().orElse(null);

        if (authorId == null || !graph.containsKey(authorId)) {
            log("Geçersiz veya mevcut olmayan yazar kimliği.");
            return;
        }

        Set<String> subgraphNodes = new HashSet<>();
        subgraphNodes.add(authorId);
        subgraphNodes.addAll(graph.get(authorId).keySet());

        Map<String, Map<String, Integer>> distances = new HashMap<>();
        for (String node : subgraphNodes) {
            distances.put(node, new HashMap<>());
            for (String target : subgraphNodes) {
                if (node.equals(target)) {
                    distances.get(node).put(target, 0);
                } else if (graph.get(node).containsKey(target)) {
                    distances.get(node).put(target, graph.get(node).get(target)); 
                } else {
                    distances.get(node).put(target, Integer.MAX_VALUE);
                }
            }
        }

        for (String k : subgraphNodes) {
            for (String i : subgraphNodes) {
                for (String j : subgraphNodes) {
                    if (distances.get(i).get(k) != Integer.MAX_VALUE && distances.get(k).get(j) != Integer.MAX_VALUE) {
                        int newDistance = distances.get(i).get(k) + distances.get(k).get(j);
                        if (newDistance < distances.get(i).get(j)) {
                            distances.get(i).put(j, newDistance);
                        }
                    }
                }
            }
        }

        log("Yazar ve ortak çalışanlar için en kısa yol mesafeleri:");
        for (String source : distances.keySet()) {
            for (String destination : distances.get(source).keySet()) {
                int dist = distances.get(source).get(destination);
                log(source + " to " + destination + " = " + (dist == Integer.MAX_VALUE ? "∞" : dist));
            }
        }

        highlightShortestPathsForSubgraph(distances, authorId);
    }

    private void highlightShortestPathsForSubgraph(Map<String, Map<String, Integer>> distances, String authorId) {
        GraphicsContext gc = graphCanvas.getGraphicsContext2D();
        gc.clearRect(0, 0, graphCanvas.getWidth(), graphCanvas.getHeight());

        double centerX = graphCanvas.getWidth() / 2;
        double centerY = graphCanvas.getHeight() / 2;
        double radius = 150;

        Map<String, double[]> tempPositions = new HashMap<>();
        tempPositions.put(authorId, new double[]{centerX, centerY});

        double angle = 0;
        for (String collaborator : graph.get(authorId).keySet()) {
            double x = centerX + radius * Math.cos(angle);
            double y = centerY + radius * Math.sin(angle);
            tempPositions.put(collaborator, new double[]{x, y});
            angle += 2 * Math.PI / graph.get(authorId).size();
        }

        for (String src : distances.keySet()) {
            for (String dest : distances.get(src).keySet()) {
                if (distances.get(src).get(dest) < Integer.MAX_VALUE && tempPositions.containsKey(src) && tempPositions.containsKey(dest)) {
                    double[] srcPos = tempPositions.get(src);
                    double[] destPos = tempPositions.get(dest);

                    gc.setStroke(javafx.scene.paint.Color.DARKGREEN);
                    gc.setLineWidth(2);
                    gc.strokeLine(srcPos[0], srcPos[1], destPos[0], destPos[1]);
                }
            }
        }

        for (String node : tempPositions.keySet()) {
            double[] pos = tempPositions.get(node);
            double radiusSize = node.equals(authorId) ? 20 : 10;

            gc.setFill(node.equals(authorId) ? javafx.scene.paint.Color.MAGENTA : javafx.scene.paint.Color.LIGHTBLUE);
            gc.fillOval(pos[0] - radiusSize, pos[1] - radiusSize, radiusSize * 2, radiusSize * 2);

            gc.setFill(javafx.scene.paint.Color.BLACK);
            gc.fillText(authorMap.getOrDefault(node, node), pos[0] + 5, pos[1] - 5);
        }
    }



    private void calculateCoauthorCount() {
        TextInputDialog dialog = new TextInputDialog();
        dialog.setTitle("Coauthor Count");
        dialog.setHeaderText("Bir yazar ID'si giriniz:");
        String authorId = dialog.showAndWait().orElse(null);

        if (authorId == null || !graph.containsKey(authorId)) {
            log("Geçersiz veya grafikte bulunmayan bir yazar ID'si girdiniz.");
            return;
        }

        int coauthorCount = graph.get(authorId).size();
        String authorName = authorMap.getOrDefault(authorId, "Bilinmeyen Yazar");

        List<String> coauthorNames = new ArrayList<>();
        for (String coauthorId : graph.get(authorId).keySet()) {
            coauthorNames.add(authorMap.getOrDefault(coauthorId, coauthorId));
        }

        log("Yazar: " + authorName + " (" + authorId + ")");
        log("İşbirliği yaptığı toplam yazar sayısı: " + coauthorCount);
        log("İşbirlikçi Yazarlar: " + String.join(", ", coauthorNames));
    }


    private void findMostCollaborativeAuthor() {
        String mostCollaborativeAuthor = null;
        int maxCollaborations = 0;

        for (String author : graph.keySet()) {
            int collaborations = graph.get(author).size();
            if (collaborations > maxCollaborations) {
                maxCollaborations = collaborations;
                mostCollaborativeAuthor = author;
            }
        }

        if (mostCollaborativeAuthor != null) {
            String authorName = authorMap.getOrDefault(mostCollaborativeAuthor, "Bilinmeyen Yazar");
            log("En işbirlikçi yazar: " + authorName + " (" + mostCollaborativeAuthor + ")");
            log("Toplam işbirlikçi sayısı: " + maxCollaborations);
        } else {
            log("Graf içinde hiçbir yazar bulunamadı.");
        }
    }


    private void findLongestPath() {
        TextInputDialog dialog = new TextInputDialog();
        dialog.setTitle("En Uzun Yol");
        dialog.setHeaderText("Bir yazar kimliği girin:");
        String authorId = dialog.showAndWait().orElse(null);

        if (authorId == null || !graph.containsKey(authorId)) {
            log("Geçersiz veya mevcut olmayan yazar kimliği.");
            return;
        }

        
        Set<String> visited = new HashSet<>();
        List<String> longestPath = new ArrayList<>();
        List<String> currentPath = new ArrayList<>();

        dfsLongestPath(authorId, visited, currentPath, longestPath);

        log("Yazardan başlayan en uzun yol: " + authorMap.getOrDefault(authorId, authorId));
        for (String node : longestPath) {
            log(authorMap.getOrDefault(node, node));
        }
        log("Yol uzunluğu: " + longestPath.size());

        drawLongestPath(longestPath);
    }

    private void drawLongestPath(List<String> path) {
        GraphicsContext gc = graphCanvas.getGraphicsContext2D();
        gc.clearRect(0, 0, graphCanvas.getWidth(), graphCanvas.getHeight()); // Tuvali temizle

        for (int i = 0; i < path.size(); i++) {
            String node = path.get(i);
            double[] pos = nodePositions.get(node);
            if (pos == null) continue;

            boolean isStartOrEnd = (i == 0 || i == path.size() - 1);
            gc.setFill(isStartOrEnd ? javafx.scene.paint.Color.GOLD : javafx.scene.paint.Color.DEEPSKYBLUE);
            gc.fillOval(pos[0] - 10, pos[1] - 10, 20, 20);

            gc.setFill(javafx.scene.paint.Color.BLACK);
            String label = authorMap.getOrDefault(node, node);
            gc.fillText(label, pos[0] + 15, pos[1]);

            if (i < path.size() - 1) {
                String nextNode = path.get(i + 1);
                double[] nextPos = nodePositions.get(nextNode);
                if (nextPos != null) {
                    gc.setStroke(javafx.scene.paint.Color.DARKBLUE);
                    gc.setLineWidth(2);
                    gc.strokeLine(pos[0], pos[1], nextPos[0], nextPos[1]);
                }
            }
        }
    }


    private void dfsLongestPath(String current, Set<String> visited, List<String> currentPath, List<String> longestPath) {
        visited.add(current);
        currentPath.add(current);

        boolean isEnd = true;
        for (String neighbor : graph.get(current).keySet()) {
            if (!visited.contains(neighbor)) {
                isEnd = false;
                dfsLongestPath(neighbor, visited, currentPath, longestPath);
            }
        }

        if (isEnd && currentPath.size() > longestPath.size()) {
            longestPath.clear();
            longestPath.addAll(currentPath);
        }

        currentPath.remove(currentPath.size() - 1);
        visited.remove(current);
    }

    private void log(String message) {
        processLog.appendText(message + "\n");
    }

    
  

    private void drawBSTNode(GraphicsContext gc, BST.Node node, double x, double y, double xOffset) {
        if (node == null) return;

        gc.setFill(javafx.scene.paint.Color.YELLOW);
        gc.fillOval(x - 15, y - 15, 30, 30); 
        gc.setStroke(javafx.scene.paint.Color.BLACK);
        gc.strokeOval(x - 15, y - 15, 30, 30); 

        gc.setFill(javafx.scene.paint.Color.BLACK);
        gc.setFont(new javafx.scene.text.Font(10)); 
        String key = node.key.length() > 10 ? node.key.substring(0, 10) + "..." : node.key; 
        gc.fillText(key, x - 20, y + 40); 

        if (node.left != null) {
            gc.setStroke(javafx.scene.paint.Color.DARKGRAY);
            gc.strokeLine(x, y, x - xOffset, y + 120); 
            drawBSTNode(gc, node.left, x - xOffset, y + 120, xOffset / 2); 
        }

        if (node.right != null) {
            gc.setStroke(javafx.scene.paint.Color.DARKGRAY);
            gc.strokeLine(x, y, x + xOffset, y + 120); 
            drawBSTNode(gc, node.right, x + xOffset, y + 120, xOffset / 2); 
        }
    }

    private void drawBST(BST bst) {
        GraphicsContext gc = graphCanvas.getGraphicsContext2D();
        gc.clearRect(0, 0, graphCanvas.getWidth(), graphCanvas.getHeight());

        if (bst.root != null) {
            drawBSTNode(gc, bst.root, graphCanvas.getWidth() / 2, 50, graphCanvas.getWidth() / 3);
        } else {
            log("Ağaç boş.");
        }
    }




    private void createAndModifyBST() {
        BST bst = new BST();

        for (String authorId : graph.keySet()) {
            bst.insert(authorId);
        }

        log("BST oluşturuldu. In-order traversal:");
        bst.inOrderTraversal();

        TextInputDialog dialog = new TextInputDialog();
        dialog.setTitle("BST Modifikasyonu");
        dialog.setHeaderText("BST'den silmek istediğiniz yazar ID'sini giriniz:");
        String authorId = dialog.showAndWait().orElse(null);

        if (authorId == null || authorId.isEmpty()) {
            log("Geçersiz giriş. İşlem iptal edildi.");
            return;
        }

        bst.delete(authorId);
        log("Yazar ID'si " + authorId + " silindi. Yeni in-order traversal:");
        bst.inOrderTraversal();

        drawBST(bst);
    }










private void highlightAuthorGraph(String authorId) {
    GraphicsContext gc = graphCanvas.getGraphicsContext2D();
    gc.clearRect(0, 0, graphCanvas.getWidth(), graphCanvas.getHeight());

    if (!graph.containsKey(authorId)) {
        log("Author ID not found in the graph.");
        return;
    }

    Set<String> highlightedNodes = new HashSet<>();
    highlightedNodes.add(authorId);
    highlightedNodes.addAll(graph.get(authorId).keySet());

    Map<String, double[]> tempPositions = new HashMap<>();
    double centerX = graphCanvas.getWidth() / 2;
    double centerY = graphCanvas.getHeight() / 2;

    tempPositions.put(authorId, new double[]{centerX, centerY}); 

    double angle = 0;
    double radius = 150; 
    for (String collaborator : graph.get(authorId).keySet()) {
        double x = centerX + radius * Math.cos(angle);
        double y = centerY + radius * Math.sin(angle);
        tempPositions.put(collaborator, new double[]{x, y});
        angle += 2 * Math.PI / graph.get(authorId).size();
    }

    for (String src : highlightedNodes) {
        for (String dest : graph.get(src).keySet()) {
            if (highlightedNodes.contains(dest)) {
                double[] srcPos = tempPositions.get(src);
                double[] destPos = tempPositions.get(dest);

                gc.setStroke(javafx.scene.paint.Color.DARKRED);
                gc.setLineWidth(2);
                gc.strokeLine(srcPos[0], srcPos[1], destPos[0], destPos[1]);
            }
        }
    }

    for (String node : highlightedNodes) {
        double[] pos = tempPositions.get(node);
        double radiusSize = node.equals(authorId) ? 20 : 10;

        gc.setFill(node.equals(authorId) ? javafx.scene.paint.Color.MAGENTA : javafx.scene.paint.Color.LIGHTBLUE);
        gc.fillOval(pos[0] - radiusSize, pos[1] - radiusSize, radiusSize * 2, radiusSize * 2);

       
        gc.setFill(javafx.scene.paint.Color.BLACK);
        gc.fillText(authorMap.getOrDefault(node, node), pos[0] + 5, pos[1] - 5);
    }
}





private void highlightGraphForAuthor() {
    TextInputDialog dialog = new TextInputDialog();
    dialog.setTitle("Yazar Grafiğini Vurgula");
    dialog.setHeaderText("Bir yazar kimliği girin:");
    String authorId = dialog.showAndWait().orElse(null);

    if (authorId == null || !graph.containsKey(authorId)) {
        log("Geçersiz veya mevcut olmayan yazar kimliği.");
        return;
    }

    log("Yazar için vurgulanmış grafik: " + authorMap.getOrDefault(authorId, authorId));
    highlightAuthorGraph(authorId);
}


class Node {
    String id;
    String name;
    List<Edge> edges;

    public Node(String id, String name) {
        this.id = id;
        this.name = name;
        this.edges = new ArrayList<>();
    }

    public void addEdge(Edge edge) {
        edges.add(edge);
    }

    @Override
    public String toString() {
        return "Node{id='" + id + "', name='" + name + "', edges=" + edges.size() + "}";
    }
}
class Edge {
    Node from;
    Node to;
    int weight;

    public Edge(Node from, Node to, int weight) {
        this.from = from;
        this.to = to;
        this.weight = weight;
    }

    @Override
    public String toString() {
        return "Edge{from=" + from.id + ", to=" + to.id + ", weight=" + weight + "}";
    }
}
class Graph {
    Map<String, Node> nodes;

    public Graph() {
        this.nodes = new HashMap<>();
    }

    public void addNode(String id, String name) {
        nodes.putIfAbsent(id, new Node(id, name));
    }

    public void addEdge(String fromId, String toId, int weight) {
        Node from = nodes.get(fromId);
        Node to = nodes.get(toId);

        if (from == null || to == null) {
            throw new IllegalArgumentException("Bir kenar eklemek için her iki düğüm de mevcut olmalıdır.");
        }

        Edge edge = new Edge(from, to, weight);
        from.addEdge(edge);
        to.addEdge(edge); 
    }

    public Node getNode(String id) {
        return nodes.get(id);
    }

    public List<Node> getAllNodes() {
        return new ArrayList<>(nodes.values());
    }

    @Override
    public String toString() {
        return "Graph{nodes=" + nodes.values() + "}";
    }
}


class BST {
    class Node {
        String key;
        Node left, right;

        Node(String key) {
            this.key = key;
        }
    }

    Node root;

    void insert(String key) {
        root = insertRec(root, key);
    }

    private Node insertRec(Node root, String key) {
        if (root == null) {
            return new Node(key);
        }
        if (key.compareTo(root.key) < 0) {
            root.left = insertRec(root.left, key);
        } else if (key.compareTo(root.key) > 0) {
            root.right = insertRec(root.right, key);
        }
        return root;
    }

    void delete(String key) {
        root = deleteRec(root, key);
    }

    private Node deleteRec(Node root, String key) {
        if (root == null) return root;

        if (key.compareTo(root.key) < 0) {
            root.left = deleteRec(root.left, key);
        } else if (key.compareTo(root.key) > 0) {
            root.right = deleteRec(root.right, key);
        } else {
            if (root.left == null) return root.right;
            if (root.right == null) return root.left;

            root.key = minValue(root.right);
            root.right = deleteRec(root.right, root.key);
        }
        return root;
    }

    private String minValue(Node root) {
        String minValue = root.key;
        while (root.left != null) {
            root = root.left;
            minValue = root.key;
        }
        return minValue;
    }

    void inOrderTraversal() {
        inOrderRec(root);
    }

    private void inOrderRec(Node root) {
        if (root != null) {
            inOrderRec(root.left);
            log(root.key);
            inOrderRec(root.right);
        }
    }
}



}
