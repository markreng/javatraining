
package com.markreng.util; 

/** 
 * @author dccs-reng 
 * @ProcName DTOraQuerySPL.jar 
 */ 

import java.sql.*; 
import java.io.*; 
//import java.util.Date; 
//import java.text.SimpleDateFormat; 

class DbmsOutput { 
        private File od; 
        private CallableStatement enable_stmt; 
        private CallableStatement disable_stmt; 
        private CallableStatement show_stmt; 
        public DbmsOutput( Connection conn, File outdata ) throws SQLException 
        { 
                od = outdata; 
            enable_stmt  = conn.prepareCall( "begin dbms_output.enable(:1); end;" ); 
            disable_stmt = conn.prepareCall( "begin dbms_output.disable; end;" ); 
            show_stmt = conn.prepareCall( 
                  "declare " + 
                  "    l_line varchar2(255); " + 
                  "    l_done number; " + 
                  "    l_buffer long; " + 
                  "begin " + 
                  "  loop " + 
                  "    exit when length(l_buffer)+255 > :maxbytes OR l_done = 1; " + 
                  "    dbms_output.get_line( l_line, l_done ); " + 
                  "    l_buffer := l_buffer || l_line || chr(10); " + 
                  "  end loop; " + 
                  " :done := l_done; " + 
                  " :buffer := l_buffer; " + 
                  "end;" ); 
        } 
        public void enable( int size ) throws SQLException 
        { 
            enable_stmt.setInt( 1, size ); 
            enable_stmt.executeUpdate(); 
        } 
        public void disable() throws SQLException 
        { 
            disable_stmt.executeUpdate(); 
        } 
        public void show() throws SQLException,IOException 
        { 
                int done = 0; 
            show_stmt.registerOutParameter( 2, java.sql.Types.INTEGER ); 
            show_stmt.registerOutParameter( 3, java.sql.Types.VARCHAR ); 
            for(;;) 
            { 
                show_stmt.setInt( 1, 32000 ); 
                show_stmt.executeUpdate(); 
                write(od, show_stmt.getString(3)); 
                if ( (done = show_stmt.getInt(2)) == 1 ) break; 
            } 
            if(done==0){done=1;} // avoid not used warning 
        } 
        public void close() throws SQLException 
        { 
            enable_stmt.close(); 
            disable_stmt.close(); 
            show_stmt.close(); 
        } 
        public void write(File wF,String mesInfo) throws IOException 
        { 
        if(wF == null){ 
            throw new IllegalStateException("dbmsoutput.write::function error, file cannot be null"); 
        } 
        try{ 
                Writer fw = new FileWriter(wF,true); 
            fw.write(mesInfo); 
            fw.flush(); 
            fw.close(); 
        } 
        catch(IOException e){} 
        } 
} 

public class DTOraQuerySPL { 
    // java -jar OraQuerySPL.jar $DBLINK $STMT $OUTDATA 
    // $DBLINK = "DBLINK_IP:DBLINK_PORT;DBLINK_SID;DBLINK_USER;DBLINK_PASS"
    // $STMT = "begin package_name.procedure_name('parameter');end;" 
    // $OUTDATA is file 
        private static String DBLINK_IP; 
    private static String DBLINK_PORT; 
        private static String DBLINK_SID; 
        private static String DBLINK_USER; 
        private static String DBLINK_PASS; 
        private static String BATCH_COMMAND; 
        private static String OUTDATA; 
        //private static SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:SS"); 
        /* 
    private static void DoWriter(File wF,String mesInfo) throws IOException { 
        if(wF == null){ 
            throw new IllegalStateException("File Can Not Be Null!"); 
        } 
        try{ 
            Writer fw = new FileWriter(wF,true); 
            fw.write(dateFormat.format(new Date())+"\t"+mesInfo); 
            fw.flush(); 
            fw.close(); 
        } 
        catch(IOException e){} 
    } 
    */ 
    
        public static void main(String[] args) throws Exception { 
                System.out.println("DTOraQuerySPL v0.5b modified by dccs-reng 20160412"); 
                if(args.length!=3){ 
                        System.out.println("Parameter error"); 
                        System.out.println("Usage: java -jar DTOraQuerySPL.jar \"IP:1521;SID;Username;Password\" \"PL-SQL Statement\" \"Output_File\""); 
                        return; 
                } 
                // Get Argument 0 DBLINK 
                String ARG_DBLINK=args[0];                 
                String[] PAR_DBLINK=ARG_DBLINK.split(";"); 
                DBLINK_IP=PAR_DBLINK[0].split(":")[0];                 
                DBLINK_PORT=PAR_DBLINK[0].split(":")[1];                 
                DBLINK_SID=PAR_DBLINK[1];                 
                DBLINK_USER=PAR_DBLINK[2];                 
                DBLINK_PASS=PAR_DBLINK[3];         
                // Check Argument 1 BATCH COMMAND 
                BATCH_COMMAND=args[1]; 
                // Check Argument 2 OUTPUT FILE 
                OUTDATA=args[2]; 
                // Show Parameter 
                System.out.println("ARG_DBLINK    = "+ARG_DBLINK); 
                System.out.println("DBLINK_IP     = "+DBLINK_IP); 
                System.out.println("DBLINK_PORT   = "+DBLINK_PORT); 
                System.out.println("DBLINK_SID    = "+DBLINK_SID);
                System.out.println("DBLINK_USER   = "+DBLINK_USER); 
                System.out.println("DBLINK_PASS   = "+DBLINK_PASS); 
                System.out.println("BATCH_COMMAND = "+BATCH_COMMAND); 
                System.out.println("OUTDATA       = "+OUTDATA); 
                if((DBLINK_IP==null)||(DBLINK_PORT==null)||(DBLINK_SID==null)||(DBLINK_USER==null)||(BATCH_COMMAND)==null||(OUTDATA)==null){ 
                        System.out.println("Parameter error"); 
                        return; 
                }else{ 
                        System.out.println("Parameter checked"); 
                } 
                File outdata = new File(OUTDATA); 
                outdata.delete(); 
                try{ 
                        DoBatch(BATCH_COMMAND, outdata); 
                        } 
                catch(SQLException e){ 
                        System.out.println(e.getMessage()); 
        } 
        } 

        private static Connection getDBConnection(){ 
                Connection dbConnection = null; 
                try{ 
            Class.forName("oracle.jdbc.driver.OracleDriver"); 
        } 
                catch(ClassNotFoundException e){ 
            System.out.println(e.getMessage()); 
        } 
                try{ 
                        dbConnection = DriverManager.getConnection("jdbc:oracle:thin:@"+DBLINK_IP+":1521:"+DBLINK_SID,DBLINK_USER,DBLINK_PASS);
                        return dbConnection; 
                        } 
                catch(SQLException e){System.out.println(e.getMessage());} 
                return dbConnection; 
        } 

    public static void DoBatch(String batchCommand, File od) throws SQLException,IOException 
    { 
        System.out.println("Process start"); 
        Connection dbConnection = null; 
        Statement stmt = null; 
        DbmsOutput dbmsOutput = null; 
        try { 
            dbConnection = getDBConnection(); 
            stmt = dbConnection.createStatement(); 
            dbmsOutput = new DbmsOutput(dbConnection, od); 
            dbmsOutput.enable(1000000); 
            // stmt.execute( "begin package.procedure('parameter'); end;" ); 
            // stmt.execute( "begin procedure_name('parameter'); end;" ); 
            stmt.execute( batchCommand ); 
            stmt.close(); 
            dbmsOutput.show(); 
        } 
        catch(SQLException e) { 
                System.out.println("Process fail"); 
                System.out.println(e.getMessage()); 
        } 
        finally { 
            if(stmt!=null){stmt.close();} 
            if(dbmsOutput!=null){dbmsOutput.close();} 
            if(dbConnection!=null){dbConnection.close();} 
            System.out.println("Process end"); 
        } 
    } 
}
'''
