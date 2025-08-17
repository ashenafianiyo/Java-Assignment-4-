# Java-Assignment-4-
Weekly planner App
package app;

import java.awt.Desktop;
import java.io.BufferedWriter;
import java.io.IOException;
import java.nio.file.*;
import java.time.*;
import java.time.format.DateTimeFormatter;
import java.time.temporal.WeekFields;
import java.util.Locale;
import java.util.Scanner;

public class Main {

    public static void main(String[] args) {
        // Defaults / flags
        String customTitle = null;         // --title "My Week"
        Integer forcedCount = null;        // --count 3..7
        String outFile = null;             // --out /path/to/file.md
        boolean openAfter = false;         // --open
        LocalDate startDate = null;        // --date YYYY-MM-DD (week of this date)

        // Parse CLI args (simple)
        for (int i = 0; i < args.length; i++) {
            switch (args[i].toLowerCase(Locale.ROOT)) {
                case "--title":
                    if (i + 1 < args.length) customTitle = args[++i];
                    break;
                case "--count":
                    if (i + 1 < args.length) {
                        try {
                            forcedCount = Integer.parseInt(args[++i]);
                        } catch (NumberFormatException ignored) {}
                    }
                    break;
                case "--out":
                    if (i + 1 < args.length) outFile = args[++i];
                    break;
                case "--open":
                    openAfter = true;
                    break;
                case "--date":
                    if (i + 1 < args.length) {
                        try {
                            startDate = LocalDate.parse(args[++i], DateTimeFormatter.ISO_LOCAL_DATE);
                        } catch (Exception ignored) {}
                    }
                    break;
                case "--help":
                case "-h":
                    printHelp();
                    return;
            }
        }

        // Determine week start (Monday by default)
        LocalDate today = (startDate != null) ? startDate : LocalDate.now();
        DayOfWeek firstDay = WeekFields.of(Locale.getDefault()).getFirstDayOfWeek();
        // normalize to Monday start for consistency in docs (common planning habit)
        LocalDate monday = today.with(DayOfWeek.MONDAY);
        LocalDate sunday = monday.plusDays(6);

        // Title and output location
        String title = (customTitle != null && !customTitle.isBlank())
                ? customTitle
                : "Weekly Planner – Week of " + monday.format(DateTimeFormatter.ISO_LOCAL_DATE);

        Path defaultDir = Paths.get(System.getProperty("user.home"), "Documents", "Weekly_Plans");
        String defaultName = monday + "_weekly_planner.md";
        Path outputPath = (outFile != null) ? Paths.get(outFile) : defaultDir.resolve(defaultName);

        // Ask user for 3–7 tasks (unless --count provided)
        Scanner in = new Scanner(System.in);
        int count = (forcedCount != null) ? forcedCount : askCount(in);

        String[] tasks = new String[count];
        System.out.println("\nPlease enter " + count + " task(s). Keep them short and action-oriented:");
        for (int i = 0; i < count; i++) {
            System.out.print("Task " + (i + 1) + ": ");
            String line = in.nextLine().trim();
            if (line.isEmpty()) {
                i--; // reprompt
                continue;
            }
            tasks[i] = line;
        }

        // Create Markdown content
        String md = buildMarkdown(title, monday, sunday, tasks);

        // Write file
        try {
            Files.createDirectories(outputPath.getParent());
            try (BufferedWriter bw = Files.newBufferedWriter(outputPath)) {
                bw.write(md);
            }
            System.out.println("\n✅ Planner created: " + outputPath.toAbsolutePath());

            if (openAfter) {
                try {
                    if (Desktop.isDesktopSupported()) {
                        Desktop.getDesktop().open(outputPath.toFile());
                    } else {
                        System.out.println("ℹ️ Desktop open not supported in this environment.");
                    }
                } catch (Exception e) {
                    System.out.println("ℹ️ Could not open file automatically: " + e.getMessage());
                }
            }
        } catch (IOException e) {
            System.out.println("❌ Failed to write file: " + e.getMessage());
        }
    }

    private static int askCount(Scanner in) {
        while (true) {
            System.out.print("\nHow many tasks this week? (3–7): ");
            String s = in.nextLine().trim();
            try {
                int n = Integer.parseInt(s);
                if (n >= 3 && n <= 7) return n;
            } catch (NumberFormatException ignored) {}
            System.out.println("Please enter a number between 3 and 7.");
        }
    }

    private static String buildMarkdown(String title, LocalDate monday, LocalDate sunday, String[] tasks) {
        DateTimeFormatter iso = DateTimeFormatter.ISO_LOCAL_DATE;
        int weekNum = monday.get(WeekFields.ISO.weekOfWeekBasedYear());
        int year = monday.get(WeekFields.ISO.weekBasedYear());

        StringBuilder sb = new StringBuilder();
        sb.append("# ").append(title).append("\n\n");
        sb.append("- **Week:** ").append(year).append("-W").append(String.format("%02d", weekNum)).append("\n");
        sb.append("- **Range:** ").append(monday.format(iso)).append(" → ").append(sunday.format(iso)).append("\n");
        sb.append("- **Generated:** ").append(LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm"))).append("\n");
        sb.append("\n---\n\n");

        sb.append("## ✅ Top Priorities (3–7 tasks)\n");
        for (String t : tasks) sb.append("- [ ] ").append(t).append("\n");

        sb.append("\n---\n\n");
        sb.append("## 🗓 Daily Notes\n");
        for (int i = 0; i < 7; i++) {
            LocalDate d = monday.plusDays(i);
            sb.append("### ").append(d.getDayOfWeek()).append(" (").append(d.format(DateTimeFormatter.ISO_LOCAL_DATE)).append(")\n");
            sb.append("- Morning: \n- Afternoon: \n- Evening: \n\n");
        }

        sb.append("---\n\n");
        sb.append("_Created with Weekly_Planner_App (Java CLI)_\n");
        return sb.toString();
    }

    private static void printHelp() {
        System.out.println("""
            Weekly Planner (Java only)

            Usage:
              java -jar weekly-planner.jar [--title "Custom Title"] [--count 3..7]
                                           [--date YYYY-MM-DD] [--out file.md] [--open]

            Options:
              --title   Custom document title
              --count   Number of tasks (3–7). If omitted, you will be prompted
              --date    Any date in the week you want to plan (defaults to today)
              --out     Output file path (default: ~/Documents/Weekly_Plans/<monday>_weekly_planner.md)
              --open    Open the generated file after creation

            Examples:
              java -jar weekly-planner.jar
              java -jar weekly-planner.jar --count 5 --open
              java -jar weekly-planner.jar --date 2025-08-18 --title "Sprint 12 Plan" --out sprint12.md
            """);
    }
}

