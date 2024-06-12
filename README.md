@Data
public class GenerateExcelSendEmailVo<T> {
 
    /**
     * 生成excel的数据
     */
    private List<T> dataList;
 
    /**
     * excel的表头
     */
    private List<String> tableHeadList;
 
    /**
     * 邮件收件人邮箱，支持多个收件人邮箱
     */
    private List<String> acceptAddressList;
 
    /**
     * 邮件的标题
     */
    private String emailTitle;
 
    /**
     * 邮件内容
     */
    private String emailContent;
}

---------------------------------

/**
 * @Author xuhongchang
 * @Date 2021/3/4  5:38 下午
 * @Describetion 项目中使用发送邮件
 */
@Component
@Slf4j
public class EmailUtil <T extends BaseRowModel>{
 
    @Autowired
    private ExcelFactory excelFactory;
 
    @Autowired
    private EmailService emailService;
 
    @Resource
    private ThreadPoolTaskExecutor threadPool;
 
    /**
     * @Author      xuhongchang
     * @Date        2021/3/5  10:47 上午
     * @Describetion 发送邮件的入口方法
     */
    public void sendEmail(GenerateExcelSendEmailVo vo) {
        Long nanoTime = System.nanoTime();
        log.info("####开始发送邮件#### : {} -- {}", vo, nanoTime);
        threadPool.execute(() -> {
            generateExcelSendEmail(vo.getDataList(), vo.getTableHeadList(), vo.getAcceptAddressList(), vo.getEmailTitle(), vo.getEmailContent());
            log.info("####发送邮件结束#### : {}", nanoTime);
        });
 
    }
 
    /**
     * @Author xuhongchang
     * @Date 2021/3/4  11:26 上午
     * @Describetion 生成excel并发送邮件
     */
    public void generateExcelSendEmail(List<T> dataList, List<String> tableHeadList, List<String> acceptAddressList, String emailTitle, String content) {
        try {
            ByteArrayOutputStream out = new ByteArrayOutputStream();
            String fileName = "email_" + new Random().nextInt(10000) + System.currentTimeMillis() + ".xlsx";
            //生成excel
            excelFactory.createExportExcel().createExcel(out, dataList, tableHeadList);
            // 发送邮件
            emailService.sendMsgFileDs(acceptAddressList, emailTitle, content, fileName, new ByteArrayInputStream(out.toByteArray()));
        } catch (Exception e) {
            log.error("发送邮件时报错：{}", e);
        }
    }
 
}

------------------------


@Component
public class ExcelFactory<T extends BaseRowModel> {
 
    public ExportExcelUtil<T> createExportExcel() {
        return new ExportExcelUtil<>();
    }
}

-----------------------------------

@Component
@Slf4j
public class EmailService {
 
    private String USER_NAME = "";
    private String PASSWORD = "";
 
 
    public void sendMsgFileDs(List<String> acceptAddressList, String title, String text, String affixName, ByteArrayInputStream inputstream) {
        Session session = assembleSession();
        Message msg = new MimeMessage(session);
        try {
            msg.setFrom(new InternetAddress(USER_NAME));
            msg.setSubject(title);
 
            Address[] addressArr = acceptAddressList(acceptAddressList);
            msg.setRecipients(Message.RecipientType.TO, addressArr);
            MimeBodyPart contentPart = (MimeBodyPart) createContent(text, inputstream, affixName);//参数为正文内容和附件流
            MimeMultipart mime = new MimeMultipart("mixed");
            mime.addBodyPart(contentPart);
            msg.setContent(mime);
            Transport.send(msg);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
 
    public Address[] acceptAddressList(List<String> acceptAddressList) {
        // 创建邮件的接收者地址，并设置到邮件消息中
        Address[] tos = new InternetAddress[acceptAddressList.size()];
        try {
            for (int i = 0; i < acceptAddressList.size(); i++) {
                tos[i] = new InternetAddress(acceptAddressList.get(i));
            }
        } catch (AddressException e) {
            e.printStackTrace();
        }
        return tos;
    }
 
    public Session assembleSession() {
 
        String host = "host"; 
        String mailStoreType = "smtp";
        String popPort = "587";
 
        final Properties props = new Properties();
 
        props.put("mail.smtp.auth", "true");
        props.put("mail.smtp.host", host);
        props.put("mail.store.protocol", mailStoreType);
        props.put("mail.smtp.port", popPort);
        //开启SSL
        props.put("mail.smtp.starttls.enable", "true");
        props.put("mail.smtp.socketFactory.port", popPort);
        props.put("mail.smtp.socketFactory.fallback", "false");
 
        Session session = Session.getDefaultInstance(props, new MyAuthenricator(USER_NAME, PASSWORD));
        return session;
    }
 
    static Part createContent(String content, ByteArrayInputStream inputstream, String affixName) {
        MimeBodyPart contentPart = null;
        try {
            contentPart = new MimeBodyPart();
            MimeMultipart contentMultipart = new MimeMultipart("related");
            MimeBodyPart htmlPart = new MimeBodyPart();
            htmlPart.setContent(content, "text/html;charset=gbk");
            contentMultipart.addBodyPart(htmlPart);
            //附件部分
            MimeBodyPart excelBodyPart = new MimeBodyPart();
            DataSource dataSource = new ByteArrayDataSource(inputstream, "application/excel");
            DataHandler dataHandler = new DataHandler(dataSource);
            excelBodyPart.setDataHandler(dataHandler);
            excelBodyPart.setFileName(MimeUtility.encodeText(affixName));
            contentMultipart.addBodyPart(excelBodyPart);
            contentPart.setContent(contentMultipart);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return contentPart;
    }
 
    //用户名密码验证，需要实现抽象类Authenticator的抽象方法PasswordAuthentication
    static class MyAuthenricator extends Authenticator {
        String u = null;
        String p = null;
 
        public MyAuthenricator(String u, String p) {
            this.u = u;
            this.p = p;
        }
        @Override
        protected PasswordAuthentication getPasswordAuthentication() {
            return new PasswordAuthentication(u, p);
        }
    }
 
}
-------------------------------------------
@Slf4j
@Configuration
public class ThreadPoolConfiguration {
 
    /**
     * 发送邮件的线程池
     * @return
     */
    @Bean("threadPool")
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        // 设置核心线程数
        executor.setCorePoolSize(20);
        // 设置最大线程数
        executor.setMaxPoolSize(128);
        // 设置队列容量
        executor.setQueueCapacity(1000);
        // 设置线程活跃时间（秒）
        executor.setKeepAliveSeconds(60);
        // 设置默认线程名称
        executor.setThreadNamePrefix("发送邮件线程-");
        // 设置拒绝策略
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        // 等待所有任务结束后再关闭线程池
        executor.setWaitForTasksToCompleteOnShutdown(true);
        return executor;
    }
}
------------------------------------
// 测试对象
@Data
public class Student {
    private String userName;
    private String address;
}
-----------------------------
----------测试----------------------
GenerateExcelSendEmailVo vo = new GenerateExcelSendEmailVo<>();
 
// 1.构建导出数据内容
List<Student> dataList = new ArrayList<>();
Student student = new Student();
student.setUserName("小许");
student.setAddress("上海市 浦东新区");
dataList.add(student);
vo.setDataList(dataList);
 
// 2.设置表头
List<String> headList = new ArrayList<>();
headList.add("姓名");
headList.add("地址");
vo.setTableHeadList(headList);
 
// 3.设置email的title
vo.setEmailTitle("测试");
 
//4.设置email的内容
vo.setEmailContent("哈喽");
 
// 5.设置收件人
List<String> acceptAddressList = new ArrayList<>();
acceptAddressList.add("xxx@163.com");
vo.setAcceptAddressList(acceptAddressList);
 
// 6.发送邮件
emailUtil.sendEmail(vo);

-------------------
// 补流程缺少的类
@Component
@Slf4j
public class ExportExcelUtil<T extends BaseRowModel> {

    public ExportExcelUtil() {
    }

    public void createExcel(ByteArrayOutputStream out, List<T> data, List<String> tableHeadList) throws IOException {
        try {
            List<List<String>> head = getExcelHead(tableHeadList);

            ExcelWriter writer = new ExcelWriter(null, out, ExcelTypeEnum.XLSX, true);
            Table table = new Table(0);
            table.setHead(head);
            Sheet sheet1 = new Sheet(1, 0);
            sheet1.setAutoWidth(true);
            sheet1.setSheetName("sheet1");
            writer.write(data, sheet1, table);
            writer.finish();
            out.flush();
        } finally {
            if (out != null) {
                out.close();
            }
        }
    }
    private List<List<String>> getExcelHead(List<String> tableHeadList){
        List<List<String>> head = new ArrayList<List<String>>();
        for (String s : tableHeadList) {
            List<String> column = new ArrayList<String>();
            column.add(s);
            head.add(column);
        }
        return head;
    }

}
