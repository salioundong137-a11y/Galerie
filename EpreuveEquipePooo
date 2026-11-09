import java.io.ByteArrayOutputStream;
import java.io.PrintStream;
import java.lang.reflect.*;
import java.util.concurrent.*;

/**
 * AUTO-CONTRÔLE de l'épreuve IsepCash — NE PAS MODIFIER.
 *
 * Lancez-le aussi souvent que vous voulez :
 *     javac -d bin src/*.java
 *     java -cp bin Verification
 * Chaque ligne indique un test, les points obtenus et ce qui ne va pas.
 * Une classe ou une méthode absente fait seulement échouer les tests qui la concernent.
 * (L'enseignant peut lancer « java -cp bin Verification 7 » pour imposer le groupe 7.)
 */
public class Verification {

    static int G;
    static long TAUX, PLAFOND;
    static final long SEUIL = 50_000;
    static final double[] TOTAL = new double[4];       // A, B, C, intégration
    static final double[] MAX = new double[4];

    // ---------------------------------------------------------------- modèle de référence
    static long fraisRef(String type, long m) {
        switch (type) {
            case "STD": return m * TAUX / 1000;
            case "PRM": return m <= SEUIL ? 0 : (m * TAUX / 1000) / 2;
            default:    return m * TAUX / 2000;
        }
    }

    static long plafondRef(String type) {
        return type.equals("PRM") ? 2 * PLAFOND : type.equals("MAR") ? 5 * PLAFOND : PLAFOND;
    }

    static String fmt(long m) {
        String s = String.valueOf(m), r = "";
        for (int i = 0; i < s.length(); i++) {
            if (i > 0 && (s.length() - i) % 3 == 0) r += " ";
            r += s.charAt(i);
        }
        return r;
    }

    static String masq(String n) {
        String r = n.substring(0, 2);
        for (int i = 2; i < n.length() - 2; i++) r += "*";
        return r + n.substring(n.length() - 2);
    }

    // ---------------------------------------------------------------- accès réflexif
    static Class<?> cls(String n) throws ClassNotFoundException { return Class.forName(n); }

    static Object compte(String type, String num, String nom) throws Exception {
        switch (type) {
            case "STD": return cls("CompteStandard").getConstructor(String.class, String.class).newInstance(num, nom);
            case "PRM": return cls("ComptePremium").getConstructor(String.class, String.class).newInstance(num, nom);
            default:    return cls("CompteMarchand").getConstructor(String.class, String.class, String.class)
                                .newInstance(num, nom, "Boutique " + nom);
        }
    }

    static Object appel(Object cible, String m, Class<?>[] types, Object... args) throws Exception {
        Method meth = (cible instanceof Class<?> c ? c : cible.getClass()).getMethod(m, types);
        try {
            return meth.invoke(cible instanceof Class<?> ? null : cible, args);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            throw (t instanceof Exception ex) ? ex : new RuntimeException(t);
        }
    }

    static Object appel(Object cible, String m) throws Exception { return appel(cible, m, new Class<?>[0]); }

    static long solde(Object c) throws Exception { return ((Number) appel(c, "getSolde")).longValue(); }
    static boolean deposer(Object c, long m) throws Exception { return (Boolean) appel(c, "deposer", new Class<?>[]{long.class}, m); }
    static boolean retirer(Object c, long m) throws Exception { return (Boolean) appel(c, "retirer", new Class<?>[]{long.class}, m); }
    static long frais(Object c, long m) throws Exception { return ((Number) appel(c, "frais", new Class<?>[]{long.class}, m)).longValue(); }

    static Object plateforme() throws Exception { return cls("Plateforme").getConstructor().newInstance(); }
    static boolean ajouter(Object p, Object c) throws Exception { return (Boolean) appel(p, "ajouter", new Class<?>[]{cls("Compte")}, c); }
    static Object rechercher(Object p, String n) throws Exception { return appel(p, "rechercher", new Class<?>[]{String.class}, n); }
    static boolean transferer(Object p, String a, String b, long m) throws Exception {
        return (Boolean) appel(p, "transferer", new Class<?>[]{String.class, String.class, long.class}, a, b, m);
    }
    static Object ussd(String m, Class<?>[] t, Object... a) throws Exception { return appel(cls("Ussd"), m, t, a); }

    // ---------------------------------------------------------------- exécution des tests
    interface Corps { double executer(StringBuilder msg) throws Exception; }

    static final ExecutorService EXEC = Executors.newCachedThreadPool(r -> {
        Thread t = new Thread(r);
        t.setDaemon(true);
        return t;
    });
    static final PrintStream OUT = sortie();

    static PrintStream sortie() {
        try {
            return new PrintStream(new java.io.FileOutputStream(java.io.FileDescriptor.out), true, "UTF-8");
        } catch (java.io.UnsupportedEncodingException e) {
            return System.out;
        }
    }

    static void test(int partie, String id, double max, String quoi, Corps corps) {
        StringBuilder msg = new StringBuilder();
        double pts;
        Future<Double> f = EXEC.submit(() -> corps.executer(msg));
        try {
            pts = f.get(4, TimeUnit.SECONDS);
        } catch (TimeoutException e) {
            f.cancel(true);
            pts = 0;
            msg.append("trop long : boucle infinie ?");
        } catch (ExecutionException e) {
            pts = 0;
            Throwable c = e.getCause();
            if (c instanceof ClassNotFoundException) msg.append("classe absente : ").append(c.getMessage());
            else if (c instanceof NoSuchMethodException) msg.append("méthode ou constructeur absent : ").append(c.getMessage());
            else msg.append(c.getClass().getSimpleName()).append(c.getMessage() == null ? "" : " : " + c.getMessage());
        } catch (InterruptedException e) {
            pts = 0;
        }
        pts = Math.max(0, Math.min(max, Math.round(pts * 4) / 4.0));
        TOTAL[partie] += pts;
        MAX[partie] += max;
        String m = msg.toString().replace('\n', ' ').trim();
        OUT.println("TEST " + id + " " + pts + " " + max + " " + (pts == max ? "ok" : m.isEmpty() ? "incomplet" : m)
                + "   [" + quoi + "]");
    }

    static double part(int bons, int total, double max) { return max * bons / total; }

    // =====================================================================================
    public static void main(String[] args) throws Exception {
        if (args.length > 0) {
            G = Integer.parseInt(args[0]);
        } else {
            try {
                G = cls("Parametres").getField("GROUPE").getInt(null);
            } catch (Exception e) {
                G = 0;
            }
        }
        TAUX = 10 + G % 6;
        PLAFOND = 150_000 + 10_000 * (G % 8);
        OUT.println("AUTO-CONTROLE IsepCash — groupe " + G + " (TAUX_FRAIS = " + TAUX + " pour mille, PLAFOND_RETRAIT = "
                + PLAFOND + " FCFA)");
        OUT.println();

        // ============================== PARTIE A : comptes ==============================
        test(0, "A1", 1.0, "hiérarchie et encapsulation", msg -> {
            double p = 0;
            Class<?> compte = cls("Compte");
            if (Modifier.isAbstract(compte.getModifiers())) p += 0.25; else msg.append("Compte doit être abstraite; ");
            boolean h = cls("CompteStandard").getSuperclass() == compte
                    && cls("ComptePremium").getSuperclass() == cls("CompteStandard")
                    && cls("CompteMarchand").getSuperclass() == compte;
            if (h) p += 0.25; else msg.append("arbre d'héritage incorrect; ");
            boolean prives = true;
            int attributs = 0;
            for (Field f : compte.getDeclaredFields()) {
                if (Modifier.isStatic(f.getModifiers())) continue;
                attributs++;
                if (!Modifier.isPrivate(f.getModifiers())) prives = false;
            }
            if (prives && attributs >= 3) p += 0.25; else msg.append("attributs de Compte non privés; ");
            boolean pasDeSetter = true;
            for (Method m : compte.getMethods()) if (m.getName().equals("setSolde")) pasDeSetter = false;
            if (pasDeSetter) p += 0.25; else msg.append("setSolde casse l'encapsulation; ");
            return p;
        });

        test(0, "A2", 1.5, "deposer / retirer", msg -> {
            int bons = 0;
            Object c = compte("STD", "771000001", "Test");
            if (solde(c) == 0) bons++; else msg.append("solde initial != 0; ");
            if (deposer(c, 100_000) && solde(c) == 100_000) bons++; else msg.append("deposer(100000) incorrect; ");
            if (!deposer(c, 0) && !deposer(c, -5) && solde(c) == 100_000) bons++; else msg.append("dépôt <= 0 accepté; ");
            if (!retirer(c, 0) && solde(c) == 100_000) bons++; else msg.append("retrait de 0 accepté; ");
            long m = 40_000;
            if (retirer(c, m) && solde(c) == 100_000 - m - fraisRef("STD", m)) bons++;
            else msg.append("retirer(40000) : solde ").append(solde(c)).append("; ");
            long avant = solde(c);
            if (!retirer(c, avant) && solde(c) == avant) bons++; else msg.append("retrait sans couvrir les frais accepté; ");
            Object riche = compte("STD", "771000002", "Riche");
            deposer(riche, 10 * PLAFOND);
            if (!retirer(riche, PLAFOND + 1) && retirer(riche, PLAFOND)) bons++; else msg.append("plafond non respecté; ");
            return part(bons, 7, 1.5);
        });

        test(0, "A3", 1.5, "frais() redéfinie (liaison dynamique)", msg -> {
            long[] montants = {1_000, 25_000, 50_000, 50_001, 120_000};
            String[] types = {"STD", "PRM", "MAR"};
            int bons = 0, total = 0;
            for (String t : types) {
                Object c = compte(t, "77100000" + t.length(), "X");
                for (long m : montants) {
                    total++;
                    if (frais(c, m) == fraisRef(t, m)) bons++;
                    else if (msg.length() < 120) msg.append(t).append(".frais(").append(m).append(")=").append(frais(c, m))
                            .append(" attendu ").append(fraisRef(t, m)).append("; ");
                }
            }
            // retirer doit utiliser les frais de la classe réelle
            Object prm = compte("PRM", "781000003", "P");
            deposer(prm, 100_000);
            if (retirer(prm, 30_000) && solde(prm) == 70_000) bons++; else msg.append("retirer n'utilise pas frais() de Premium; ");
            total++;
            return part(bons, total, 1.5);
        });

        test(0, "A4", 0.5, "plafond() redéfinie", msg -> {
            int bons = 0;
            for (String t : new String[]{"STD", "PRM", "MAR"}) {
                long p = ((Number) appel(compte(t, "770000009", "X"), "plafond")).longValue();
                if (p == plafondRef(t)) bons++; else msg.append(t).append(" plafond ").append(p).append("; ");
            }
            return part(bons, 3, 0.5);
        });

        test(0, "A5", 1.0, "toString", msg -> {
            int bons = 0;
            Object s = compte("STD", "771234567", "Awa Diop");
            deposer(s, 12_500);
            Object pr = compte("PRM", "781112233", "Moussa Fall");
            Object ma = compte("MAR", "701234500", "Fatou Sarr");
            String[][] cas = {
                {s.toString(), "[STD] 771234567 Awa Diop 12500 FCFA"},
                {pr.toString(), "[PRM] 781112233 Moussa Fall 0 FCFA"},
                {ma.toString(), "[MAR] 701234500 Fatou Sarr 0 FCFA (Boutique Fatou Sarr)"}};
            for (String[] c : cas) {
                if (c[0].equals(c[1])) bons++;
                else if (msg.length() < 140) msg.append("attendu «").append(c[1]).append("» obtenu «").append(c[0]).append("»; ");
            }
            return part(bons, 3, 1.0);
        });

        test(0, "A6", 0.5, "equals, hashCode, compteur static", msg -> {
            double p = 0;
            Object a = compte("STD", "775550000", "A");
            Object b = compte("PRM", "775550000", "B");
            Object c = compte("STD", "775550001", "A");
            if (a.equals(b) && !a.equals(c) && !a.equals(null) && !a.equals("775550000") && a.hashCode() == b.hashCode()) p += 0.25;
            else msg.append("equals/hashCode : deux comptes de même numéro doivent être égaux; ");
            int avant = ((Number) appel(cls("Compte"), "getNombreCrees")).intValue();
            compte("MAR", "775550002", "C");
            compte("STD", "775550003", "D");
            int apres = ((Number) appel(cls("Compte"), "getNombreCrees")).intValue();
            if (apres - avant == 2) p += 0.25; else msg.append("getNombreCrees ne compte pas les créations; ");
            return p;
        });

        // ============================== PARTIE B : plateforme ==============================
        test(1, "B1", 1.5, "ajouter / rechercher / capacité", msg -> {
            int bons = 0;
            Object p = plateforme();
            Object a = compte("STD", "771111111", "A");
            if (ajouter(p, a)) bons++; else msg.append("ajout refusé; ");
            if (!ajouter(p, compte("PRM", "771111111", "Doublon"))) bons++; else msg.append("doublon accepté; ");
            if (rechercher(p, "771111111") == a) bons++; else msg.append("rechercher ne retrouve pas le compte; ");
            if (rechercher(p, "779999999") == null) bons++; else msg.append("rechercher d'un absent != null; ");
            for (int i = 2; i <= 20; i++) ajouter(p, compte("STD", "7711111" + (10 + i), "X"));
            int n = ((Number) appel(p, "getNombre")).intValue();
            if (n == 20 && !ajouter(p, compte("STD", "771119999", "21e"))) bons++; else msg.append("capacité de 20 non respectée; ");
            if (!ajouter(p, null)) bons++; else msg.append("ajouter(null) accepté; ");
            return part(bons, 6, 1.5);
        });

        test(1, "B2", 1.5, "transferer", msg -> {
            int bons = 0;
            Object p = plateforme();
            Object a = compte("STD", "772000001", "A"), b = compte("MAR", "702000002", "B");
            ajouter(p, a);
            ajouter(p, b);
            deposer(a, 100_000);
            long f = fraisRef("STD", 30_000);
            if (transferer(p, "772000001", "702000002", 30_000) && solde(a) == 70_000 - f && solde(b) == 30_000) bons++;
            else msg.append("transfert de 30000 : soldes ").append(solde(a)).append(" / ").append(solde(b)).append("; ");
            if (((Number) appel(p, "getTotalFrais")).longValue() == f) bons++; else msg.append("total des frais incorrect; ");
            long sa = solde(a), sb = solde(b);
            boolean refus = !transferer(p, "772000001", "799999999", 1_000) && !transferer(p, "772000001", "772000001", 1_000)
                    && !transferer(p, "772000001", "702000002", 0) && !transferer(p, "772000001", "702000002", sa);
            if (refus && solde(a) == sa && solde(b) == sb) bons++; else msg.append("un transfert invalide a été accepté ou a modifié un solde; ");
            return part(bons, 3, 1.5);
        });

        test(1, "B3", 1.0, "journal des opérations", msg -> {
            Object p = plateforme();
            Object a = compte("PRM", "783000001", "A"), b = compte("STD", "773000002", "B");
            ajouter(p, a);
            ajouter(p, b);
            deposer(a, 500_000);
            transferer(p, "783000001", "773000002", 20_000);
            transferer(p, "783000001", "773000002", 80_000);
            transferer(p, "783000001", "700000000", 5_000);
            String[] j = (String[]) appel(p, "journal");
            String[] att = {"TX0001 783000001 -> 773000002 20000 FCFA (frais 0)",
                            "TX0002 783000001 -> 773000002 80000 FCFA (frais " + fraisRef("PRM", 80_000) + ")"};
            int bons = 0;
            if (j.length == 2) bons++; else msg.append("journal de taille ").append(j.length).append(" au lieu de 2; ");
            for (int i = 0; i < Math.min(2, j.length); i++)
                if (att[i].equals(j[i])) bons++; else if (msg.length() < 140) msg.append("attendu «").append(att[i]).append("» obtenu «").append(j[i]).append("»; ");
            if (j.length > 0) j[0] = "modifié";
            String[] j2 = (String[]) appel(p, "journal");
            if (j2.length > 0 && att[0].equals(j2[0])) bons++; else msg.append("journal() doit renvoyer une copie; ");
            return part(bons, 4, 1.0);
        });

        test(1, "B4", 0.5, "compterParType (instanceof)", msg -> {
            Object p = plateforme();
            ajouter(p, compte("STD", "774000001", "A"));
            ajouter(p, compte("PRM", "784000002", "B"));
            ajouter(p, compte("PRM", "784000003", "C"));
            ajouter(p, compte("MAR", "704000004", "D"));
            ajouter(p, compte("STD", "774000005", "E"));
            int[] r = (int[]) appel(p, "compterParType");
            if (java.util.Arrays.equals(r, new int[]{2, 2, 1})) return 0.5;
            msg.append("obtenu ").append(java.util.Arrays.toString(r)).append(" attendu [2, 2, 1] (Premium à tester avant Standard)");
            return 0;
        });

        test(1, "B5", 1.0, "trierParSolde (tri par insertion)", msg -> {
            Object p = plateforme();
            String[] nums = {"775000005", "775000001", "785000002", "705000003", "775000004", "765000006"};
            long[] soldes = {5_000, 90_000, 5_000, 120_000, 0, 90_000};
            for (int i = 0; i < nums.length; i++) {
                Object c = compte(i % 3 == 0 ? "STD" : i % 3 == 1 ? "PRM" : "MAR", nums[i], "X");
                deposer(c, soldes[i]);
                ajouter(p, c);
            }
            appel(p, "trierParSolde");
            Object[] cs = (Object[]) appel(p, "comptes");
            String[] att = {"705000003", "765000006", "775000001", "775000005", "785000002", "775000004"};
            int bons = 0;
            for (int i = 0; i < att.length && i < cs.length; i++)
                if (att[i].equals(appel(cs[i], "getNumero"))) bons++;
            if (bons < att.length) msg.append(bons).append("/6 positions justes (solde décroissant, puis numéro croissant)");
            return bons == att.length ? 1.0 : 0.5 * bons / att.length;
        });

        test(1, "B6", 0.5, "comptes() et comptesAuDessus() : copies de la bonne taille", msg -> {
            double pts = 0;
            Object p = plateforme();
            for (int i = 0; i < 4; i++) {
                Object c = compte("STD", "77600000" + i, "X");
                deposer(c, 10_000 * (i + 1));
                ajouter(p, c);
            }
            Object[] cs = (Object[]) appel(p, "comptes");
            if (cs.length == 4) {
                cs[0] = null;
                Object[] cs2 = (Object[]) appel(p, "comptes");
                if (cs2[0] != null) pts += 0.25; else msg.append("comptes() expose le tableau interne; ");
            } else msg.append("comptes() de taille ").append(cs.length).append("; ");
            Object[] au = (Object[]) appel(p, "comptesAuDessus", new Class<?>[]{long.class}, 30_000L);
            if (au.length == 2) pts += 0.25; else msg.append("comptesAuDessus(30000) de taille ").append(au.length).append(" au lieu de 2; ");
            return pts;
        });

        // ============================== PARTIE C : chaînes ==============================
        Class<?>[] S = {String.class};
        test(2, "C1", 1.0, "numeroValide", msg -> {
            Object[][] cas = {{"771234567", true}, {"701234567", true}, {"751234567", true}, {"761234567", true},
                {"781234567", true}, {"331234567", false}, {"79123456", false}, {"7712345678", false},
                {"77123456a", false}, {"", false}, {null, false}};
            int bons = 0;
            for (Object[] c : cas) {
                boolean r = (Boolean) ussd("numeroValide", S, c[0]);
                if (r == (Boolean) c[1]) bons++; else if (msg.length() < 120) msg.append(c[0]).append(" → ").append(r).append("; ");
            }
            return part(bons, cas.length, 1.0);
        });

        test(2, "C2", 1.0, "normaliser", msg -> {
            String[][] cas = {{"77 123 45 67", "771234567"}, {"+221 78-111-22-33", "781112233"}, {"00221701234500", "701234500"},
                {"76.101.01.01", "761010101"}, {"7712", null}, {"33 820 00 00", null}, {"+22177123456X", null}, {null, null}};
            int bons = 0;
            for (String[] c : cas) {
                Object r = ussd("normaliser", S, c[0]);
                if (c[1] == null ? r == null : c[1].equals(r)) bons++;
                else if (msg.length() < 120) msg.append("«").append(c[0]).append("» → ").append(r).append("; ");
            }
            return part(bons, cas.length, 1.0);
        });

        test(2, "C3", 0.5, "masquer", msg -> {
            String[][] cas = {{"771234567", "77*****67"}, {"781112233", "78*****33"}, {"12345", "12*45"}};
            int bons = 0;
            for (String[] c : cas) {
                Object r = ussd("masquer", S, c[0]);
                if (c[1].equals(r)) bons++; else msg.append(c[0]).append(" → ").append(r).append("; ");
            }
            return part(bons, cas.length, 0.5);
        });

        test(2, "C4", 1.0, "lireMontant (sans parseLong)", msg -> {
            Object[][] cas = {{"5000", 5000L}, {"0", 0L}, {"999999999", 999_999_999L}, {"120000", 120_000L},
                {"", -1L}, {"5O00", -1L}, {"-5", -1L}, {"1234567890", -1L}, {"12 5", -1L}, {null, -1L}};
            int bons = 0;
            for (Object[] c : cas) {
                long r = ((Number) ussd("lireMontant", S, c[0])).longValue();
                if (r == (Long) c[1]) bons++; else if (msg.length() < 120) msg.append("«").append(c[0]).append("» → ").append(r).append("; ");
            }
            return part(bons, cas.length, 1.0);
        });

        test(2, "C5", 1.0, "formaterMontant", msg -> {
            long[] cas = {0, 7, 999, 1_000, 25_000, 1_250_000, 1_000_000_000L};
            int bons = 0;
            for (long m : cas) {
                Object r = ussd("formaterMontant", new Class<?>[]{long.class}, m);
                if (fmt(m).equals(r)) bons++; else if (msg.length() < 120) msg.append(m).append(" → «").append(r).append("»; ");
            }
            return part(bons, cas.length, 1.0);
        });

        test(2, "C6", 1.5, "decouper", msg -> {
            Object[][] cas = {
                {"*144*1*771234567*5000#", new String[]{"144", "1", "771234567", "5000"}},
                {"*144*2#", new String[]{"144", "2"}},
                {"*144*1*78 111 22 33*25000#", new String[]{"144", "1", "78 111 22 33", "25000"}},
                {"*144#", new String[]{"144"}},
                {"144*2#", null}, {"*144*2", null}, {"*144**2#", null}, {"*144*2#3#", null}, {"*#", null}, {null, null}};
            int bons = 0;
            for (Object[] c : cas) {
                Object r = ussd("decouper", S, c[0]);
                boolean ok = c[1] == null ? r == null : (r instanceof String[] t && java.util.Arrays.equals(t, (String[]) c[1]));
                if (ok) bons++;
                else if (msg.length() < 120) msg.append("«").append(c[0]).append("» → ")
                        .append(r instanceof String[] t ? java.util.Arrays.toString(t) : String.valueOf(r)).append("; ");
            }
            return part(bons, cas.length, 1.5);
        });

        // ============================== INTÉGRATION ==============================
        test(3, "I1", 1.5, "Ussd.executer", msg -> {
            Object p = plateforme();
            Object a = compte("STD", "771234567", "A"), b = compte("PRM", "781112233", "B");
            ajouter(p, a);
            ajouter(p, b);
            deposer(a, 100_000);
            Class<?>[] t = {cls("Plateforme"), String.class, String.class};
            long f = fraisRef("STD", 25_000);
            String[][] cas = {
                {"771234567", "*144*1*78 111 22 33*25000#", "Transfert de 25 000 FCFA vers 78*****33 effectue. Frais : " + fmt(f) + " FCFA"},
                {"771234567", "*144*2#", "Solde : " + fmt(100_000 - 25_000 - f) + " FCFA"},
                {"771234567", "*144*1*781112233*900000#", "Transfert refuse"},
                {"771234567", "*144*1*781112233*9x0#", "Montant invalide"},
                {"771234567", "*144*1*7812*100#", "Numero invalide"},
                {"771234567", "*145*1#", "Service inconnu"},
                {"771234567", "*144*9#", "Service inconnu"},
                {"771234567", "144*2#", "Code invalide"},
                {"771234567", "*144*1*781112233#", "Code invalide"},
                {"799999999", "*144*2#", "Compte inconnu"}};
            int bons = 0;
            for (String[] c : cas) {
                Object r = ussd("executer", t, p, c[0], c[1]);
                if (c[2].equals(r)) bons++;
                else if (msg.length() < 160) msg.append(c[1]).append(" → «").append(r).append("»; ");
            }
            return part(bons, cas.length, 1.5);
        });

        test(3, "I2", 0.5, "affichage de Main (scénario du sujet)", msg -> {
            ByteArrayOutputStream tampon = new ByteArrayOutputStream();
            PrintStream ancien = System.out;
            System.setOut(new PrintStream(tampon, true, "UTF-8"));
            try {
                appel(cls("Main"), "main", new Class<?>[]{String[].class}, (Object) new String[0]);
            } finally {
                System.setOut(ancien);
            }
            String[] lignes = tampon.toString("UTF-8").strip().split("\\R");
            String[] att = scenario();
            int bons = 0;
            for (int i = 0; i < att.length; i++)
                if (i < lignes.length && lignes[i].strip().equals(att[i])) bons++;
            if (bons < att.length) msg.append(bons).append("/").append(att.length).append(" lignes conformes");
            return part(bons, att.length, 0.5);
        });

        // ------------------------------------------------------------------ bilan
        OUT.println();
        String[] noms = {"Partie A (comptes)", "Partie B (plateforme)", "Partie C (chaînes)", "Intégration"};
        double t = 0, m = 0;
        for (int i = 0; i < 4; i++) {
            OUT.printf("BILAN %-22s %5.2f / %.0f%n", noms[i], TOTAL[i], MAX[i]);
            t += TOTAL[i];
            m += MAX[i];
        }
        OUT.printf("BILAN %-22s %5.2f / %.0f%n", "TOTAL", t, m);
        System.exit(0);
    }

    static String[] scenario() {
        long f1 = fraisRef("STD", 25_000), f2 = fraisRef("STD", 120_000), f3 = fraisRef("PRM", 60_000);
        long awa = 200_000 - 25_000 - f1 - 120_000 - f2;
        long moussa = 300_000 + 25_000 - 60_000 - f3;
        return new String[]{
            "Transfert de 25 000 FCFA vers 78*****33 effectue. Frais : " + fmt(f1) + " FCFA",
            "Transfert de 120 000 FCFA vers 70*****00 effectue. Frais : " + fmt(f2) + " FCFA",
            "Solde : " + fmt(awa) + " FCFA",
            "Transfert refuse",
            "Montant invalide",
            "Service inconnu",
            "Numero invalide",
            "Transfert de 60 000 FCFA vers 70*****00 effectue. Frais : " + fmt(f3) + " FCFA",
            "[PRM] 781112233 Moussa Fall " + moussa + " FCFA",
            "[MAR] 701234500 Fatou Sarr 180000 FCFA (Boutique Keur Fatou)",
            "[STD] 771234567 Awa Diop " + awa + " FCFA",
            "[STD] 761010101 Ibrahima Ndiaye 15000 FCFA",
            "TX0001 771234567 -> 781112233 25000 FCFA (frais " + f1 + ")",
            "TX0002 771234567 -> 701234500 120000 FCFA (frais " + f2 + ")",
            "TX0003 781112233 -> 701234500 60000 FCFA (frais " + f3 + ")",
            "Frais collectes : " + fmt(f1 + f2 + f3) + " FCFA"};
    }
}
