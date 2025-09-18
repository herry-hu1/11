#include "audittrail.h"
#include "widget.h"
#include "login.h"
#include <QApplication>
#ifdef Q_OS_WIN
#include "windows.h"
#endif
#define DEBUGMODE
#undef DEBUGMODE
QString g_dataPath;
QString g_pdfPath;
QString g_dataOut;
QString g_version;
QString g_dataBase;
QString g_distIP;
QString g_localIP;
QString g_user;
QString g_pwd;
int g_authority=0;
int g_permissions=1;
bool g_ncs;
int g_cr;
int main(int argc, char *argv[])
{
    QApplication a(argc, argv);
    QString currentpath = QDir::currentPath();
    g_dataPath = currentpath+"/data/";
    g_pdfPath = currentpath+"/pdf/";
    g_dataOut = currentpath+"/dataout/";
   
    QSettings setini("config.ini",QSettings::IniFormat);
    g_ncs = setini.value("Param/ncs",0).toBool();
    g_cr = setini.value("Param/cr",100).toInt();
    g_dataBase = setini.value("SYS/db","trxdata_t").toString();
    g_distIP = setini.value("SYS/dist","192.168.11.11").toString();
    g_localIP = setini.value("SYS/local","192.168.11.100").toString();
    g_permissions = setini.value("SYS/p",1).toInt();
#ifdef Q_OS_WIN
    if(g_permissions==2){
        AllocConsole();
        freopen("CONOUT$", "w", stdout);
        freopen("CONOUT$", "w", stderr);
    }
#endif
#ifdef DEBUGMODE
    Widget w;
    w.show();
    return a.exec();
#else
    Login w;
    if(w.exec()==QDialog::Accepted){
        AuditTrail  m;
        m.show();
        return a.exec();
    }
    else
        return 0;
#endif
}

#ifndef AUDITTRAIL_H
#define AUDITTRAIL_H

#include <QWidget>

namespace Ui {
class AuditTrail;
}

class AuditTrail : public QWidget
{
    Q_OBJECT

public:
    explicit AuditTrail(QWidget *parent = nullptr);
    ~AuditTrail();

private slots:
    void on_btn_query_clicked();
    void onSectionClicked(int index);
    void on_cb_time_stateChanged(int arg1);

    void on_cb_type_stateChanged(int arg1);

    void on_cb_key_stateChanged(int arg1);

    void on_btn_out_clicked();

private:
    Ui::AuditTrail *ui;
    QString lastqry;
    QString orderby="ORDER BY `dt`";
    int uorder=1;
};

#endif // AUDITTRAIL_H

#include "audittrail.h"
#include "ui_audittrail.h"
#include "databasethread.h"
#include <QDebug>
#include <QFileDialog>
#include "xlsxdocument.h"
#include "xlsxworkbook.h"
#include "xlsxworksheet.h"
AuditTrail::AuditTrail(QWidget *parent) :
    QWidget(parent),
    ui(new Ui::AuditTrail)
{
    ui->setupUi(this);
    setWindowTitle("审计追踪");
    ui->tableView->verticalHeader()->setVisible(true);
    ui->tableView->setAlternatingRowColors(true);
    ui->tableView->horizontalHeader()->setSectionResizeMode(QHeaderView::ResizeToContents);
    ui->tableView->horizontalHeader()->setVisible(true);
    //ui->tableView->horizontalHeader()->setStretchLastSection(true);
    //ui->tableView->setHorizontalScrollBarPolicy(Qt::ScrollBarAlwaysOff);
    ui->tableView->setEditTriggers(QAbstractItemView::NoEditTriggers);
    ui->tableView->setShowGrid(true);
    ui->tableView->setFocusPolicy(Qt::NoFocus);
    ui->tableView->setSelectionBehavior(QAbstractItemView::SelectRows);
    QHeaderView* horheader = ui->tableView->horizontalHeader();
    connect(horheader,&QHeaderView::sectionClicked,this,&AuditTrail::onSectionClicked);

    QStringList lst = DatabaseThread::Instance()->getUserList(true);
    ui->cb_user->addItem("全部");
    foreach(QString str,lst){
        ui->cb_user->addItem(str.mid(0,str.indexOf("|")));
    }
    ui->dt_ks->setDateTime(QDateTime::currentDateTime());
    ui->dt_js->setDateTime(QDateTime::currentDateTime());
}

AuditTrail::~AuditTrail()
{
    delete ui;
}

void AuditTrail::on_btn_query_clicked()
{
    QString sqltext="SELECT `dt`,`user`,`type`,`msg`,`oldvalue`,`newvalue` FROM `logs` WHERE ";
    QString condition;
    if(ui->cb_user->currentIndex()>0)
        condition.append("AND `user`='"+ui->cb_user->currentText()+"' ");
    if(ui->cb_time->isChecked()){
        QString ks = ui->dt_ks->dateTime().toString("yyyy-MM-dd hh:mm:ss");
        QString js = ui->dt_js->dateTime().toString("yyyy-MM-dd hh:mm:ss");
        condition.append("AND `dt`>='"+ks+"' AND `dt`<='"+js+"' ");
    }
    if(ui->cb_type->isChecked()){
        condition.append("AND `type` IN ("+ui->edt_type->text()+") ");
    }
    if(ui->cb_key->isChecked()){
        condition.append("AND `msg` LIKE '%"+ui->edt_key->text()+"%' ");
    }
    QString sql;
    if(condition.isEmpty()){
        sqltext.remove("WHERE");
        sql=sqltext+orderby;
        lastqry=sqltext;
    }
    else{
        condition.remove(0,3);
        sql=sqltext+condition+orderby;
        lastqry=sqltext+condition;
    }
    qDebug()<<sql;
    ui->tableView->setModel(DatabaseThread::Instance()->getAudittrail(sql,uorder));
}

void AuditTrail::onSectionClicked(int index)
{
    if(index>2)
        return;
    if(abs(uorder)==index+1)
        uorder = -uorder;
    else
        uorder = index+1;
    switch (uorder) {
    case -1:
        orderby = " ORDER BY `dt` DESC";
        break;
    case 1:
        orderby = " ORDER BY `dt`";
        break;
    case -2:
        orderby = " ORDER BY `user` DESC";
        break;
    case 2:
        orderby = " ORDER BY `user`";
        break;
    case -3:
        orderby = " ORDER BY `type` DESC";
        break;
    case 3:
        orderby = " ORDER BY `type`";
        break;
    }
    ui->tableView->setModel(DatabaseThread::Instance()->getAudittrail(lastqry+orderby,uorder));
}

void AuditTrail::on_cb_time_stateChanged(int arg1)
{
    if(arg1==0){
        ui->dt_ks->setEnabled(false);
        ui->dt_js->setEnabled(false);
    }
    else if(arg1==2){
        ui->dt_ks->setEnabled(true);
        ui->dt_js->setEnabled(true);
    }
}

void AuditTrail::on_cb_type_stateChanged(int arg1)
{
    if(arg1==0){
        ui->edt_type->setEnabled(false);
    }
    else if(arg1==2){
        ui->edt_type->setEnabled(true);
    }
}

void AuditTrail::on_cb_key_stateChanged(int arg1)
{
    if(arg1==0){
        ui->edt_key->setEnabled(false);
    }
    else if(arg1==2){
        ui->edt_key->setEnabled(true);
    }
}

void AuditTrail::on_btn_out_clicked()
{
    QString savepath=QFileDialog::getSaveFileName(this,"导出至","./","*.xlsx");
    if(savepath.isEmpty())
        return;
    QAbstractItemModel *model = ui->tableView->model();
    if(!model)
        return;
    QXlsx::Document xlsx;
    xlsx.addSheet("审计");
    xlsx.selectSheet("审计");
    for(int col=0;col<model->columnCount();col++){
        xlsx.write(1,col+1,model->headerData(col,Qt::Horizontal).toString());
    }
    for(int row=0;row<model->rowCount();row++){
        for(int col=0;col<model->columnCount();col++){
            QModelIndex index = model->index(row,col);
            xlsx.write(row+2,col+1,model->data(index).toString());
        }
    }
    xlsx.saveAs(savepath);
}

<?xml version="1.0" encoding="UTF-8"?>
<ui version="4.0">
 <class>AuditTrail</class>
 <widget class="QWidget" name="AuditTrail">
  <property name="geometry">
   <rect>
    <x>0</x>
    <y>0</y>
    <width>550</width>
    <height>657</height>
   </rect>
  </property>
  <property name="windowTitle">
   <string>Form</string>
  </property>
  <layout class="QVBoxLayout" name="verticalLayout">
   <item>
    <layout class="QHBoxLayout" name="horizontalLayout" stretch="0,1,0,0,1,0,1">
     <property name="spacing">
      <number>6</number>
     </property>
     <item>
      <widget class="QLabel" name="label">
       <property name="text">
        <string>用户名：</string>
       </property>
      </widget>
     </item>
     <item>
      <widget class="QComboBox" name="cb_user">
       <property name="maximumSize">
        <size>
         <width>140</width>
         <height>16777215</height>
        </size>
       </property>
      </widget>
     </item>
     <item>
      <spacer name="horizontalSpacer">
       <property name="orientation">
        <enum>Qt::Horizontal</enum>
       </property>
       <property name="sizeHint" stdset="0">
        <size>
         <width>40</width>
         <height>20</height>
        </size>
       </property>
      </spacer>
     </item>
     <item>
      <widget class="QCheckBox" name="cb_time">
       <property name="text">
        <string>起止时间：</string>
       </property>
      </widget>
     </item>
     <item>
      <widget class="QDateTimeEdit" name="dt_ks">
       <property name="enabled">
        <bool>false</bool>
       </property>
       <property name="maximumSize">
        <size>
         <width>160</width>
         <height>16777215</height>
        </size>
       </property>
       <property name="displayFormat">
        <string>yyyy-MM-dd HH:mm:ss</string>
       </property>
       <property name="calendarPopup">
        <bool>true</bool>
       </property>
      </widget>
     </item>
     <item>
      <widget class="QLabel" name="label_4">
       <property name="text">
        <string>-</string>
       </property>
       <property name="alignment">
        <set>Qt::AlignCenter</set>
       </property>
      </widget>
     </item>
     <item>
      <widget class="QDateTimeEdit" name="dt_js">
       <property name="enabled">
        <bool>false</bool>
       </property>
       <property name="maximumSize">
        <size>
         <width>160</width>
         <height>16777215</height>
        </size>
       </property>
       <property name="displayFormat">
        <string>yyyy-MM-dd HH:mm:ss</string>
       </property>
       <property name="calendarPopup">
        <bool>true</bool>
       </property>
      </widget>
     </item>
    </layout>
   </item>
   <item>
    <layout class="QHBoxLayout" name="horizontalLayout_2" stretch="0,1,0,0,0,1">
     <item>
      <widget class="QCheckBox" name="cb_type">
       <property name="text">
        <string>事件类型：</string>
       </property>
      </widget>
     </item>
     <item>
      <widget class="QLineEdit" name="edt_type">
       <property name="enabled">
        <bool>false</bool>
       </property>
       <property name="maximumSize">
        <size>
         <width>140</width>
         <height>16777215</height>
        </size>
       </property>
       <property name="toolTip">
        <string notr="true">&lt;html&gt;&lt;head/&gt;&lt;body&gt;&lt;p&gt;0.登录&lt;/p&gt;&lt;p&gt;1.新增任务&lt;/p&gt;&lt;p&gt;2.更新任务&lt;/p&gt;&lt;p&gt;3.删除任务&lt;/p&gt;&lt;p&gt;4.开始任务&lt;/p&gt;&lt;p&gt;5.完成任务&lt;/p&gt;&lt;p&gt;6.新增核素库&lt;/p&gt;&lt;p&gt;7.更新核素库&lt;/p&gt;&lt;p&gt;8.删除核素库&lt;/p&gt;&lt;p&gt;9.新增无淬灭库&lt;/p&gt;&lt;p&gt;10.更新无淬灭库&lt;/p&gt;&lt;p&gt;11.删除无淬灭库&lt;/p&gt;&lt;p&gt;12.新增系列淬灭库&lt;/p&gt;&lt;p&gt;13.更新系列淬灭库&lt;/p&gt;&lt;p&gt;14.删除系列淬灭库&lt;/p&gt;&lt;p&gt;15.新增标样库&lt;/p&gt;&lt;p&gt;16.更新标样库&lt;/p&gt;&lt;p&gt;17.删除标样库&lt;/p&gt;&lt;p&gt;18.新增ab库&lt;/p&gt;&lt;p&gt;19.更新ab库&lt;/p&gt;&lt;p&gt;20.删除ab库&lt;/p&gt;&lt;p&gt;21.删除历史数据&lt;/p&gt;&lt;p&gt;22.重计算历史数据&lt;/p&gt;&lt;p&gt;23.保存曲线&lt;/p&gt;&lt;p&gt;24.新增本底&lt;/p&gt;&lt;p&gt;25.删除本底&lt;/p&gt;&lt;p&gt;26.更新本底备注&lt;/p&gt;&lt;p&gt;27.删除历史检验数据&lt;/p&gt;&lt;p&gt;28.删除ID数据&lt;/p&gt;&lt;p&gt;29.重计算ID数据&lt;/p&gt;&lt;p&gt;30.更新ID数据窗口&lt;/p&gt;&lt;p&gt;40.更改窗口&lt;/p&gt;&lt;p&gt;&lt;br/&gt;&lt;/p&gt;&lt;/body&gt;&lt;/html&gt;</string>
       </property>
       <property name="placeholderText">
        <string>例:0,1,2</string>
       </property>
      </widget>
     </item>
     <item>
      <widget class="QLabel" name="label_3">
       <property name="text">
        <string>(多编号用半角逗号分隔)</string>
       </property>
      </widget>
     </item>
     <item>
      <spacer name="horizontalSpacer_2">
       <property name="orientation">
        <enum>Qt::Horizontal</enum>
       </property>
       <property name="sizeHint" stdset="0">
        <size>
         <width>40</width>
         <height>20</height>
        </size>
       </property>
      </spacer>
     </item>
     <item>
      <widget class="QCheckBox" name="cb_key">
       <property name="text">
        <string>关键字：</string>
       </property>
      </widget>
     </item>
     <item>
      <widget class="QLineEdit" name="edt_key">
       <property name="enabled">
        <bool>false</bool>
       </property>
       <property name="maximumSize">
        <size>
         <width>140</width>
         <height>16777215</height>
        </size>
       </property>
      </widget>
     </item>
    </layout>
   </item>
   <item>
    <widget class="QTableView" name="tableView"/>
   </item>
   <item>
    <layout class="QHBoxLayout" name="horizontalLayout_3">
     <property name="spacing">
      <number>40</number>
     </property>
     <property name="leftMargin">
      <number>100</number>
     </property>
     <property name="rightMargin">
      <number>100</number>
     </property>
     <item>
      <widget class="QPushButton" name="btn_query">
       <property name="text">
        <string>查询</string>
       </property>
      </widget>
     </item>
     <item>
      <widget class="QPushButton" name="btn_out">
       <property name="text">
        <string>导出</string>
       </property>
      </widget>
     </item>
    </layout>
   </item>
  </layout>
 </widget>
 <resources/>
 <connections/>
</ui>

#ifndef DATABASETHREAD_H
#define DATABASETHREAD_H

#include <QThread>
#include <QSqlDatabase>
#include <QSqlQuery>
#include <QSqlRecord>
#include "myquerymodel.h"
#include <QSqlError>
#include <QVariant>
#include <QDate>
#include "ParamStruct.h"
#include <QMutexLocker>
class DatabaseThread : public QObject
{
    Q_OBJECT
public:

    static DatabaseThread *Instance(){
        if(!s_DatabaseThread){
            s_DatabaseThread = new DatabaseThread;
        }
        return s_DatabaseThread;
    }
    static void deleteInstance(){        
        if(s_DatabaseThread){
            s_DatabaseThread->deleteLater();
            s_DatabaseThread = Q_NULLPTR;
        }
}
void initDatabase();
int userLogin(QString user,QString pwd);
QStringList getUserList(bool containAdmin);
void updateUser(QString uname,QString pwd,bool fzr,bool mode);
void updateUser(QString uname,QString pwd);
void deleteUser(QString uname);
void lockUser(QString uname);
QSqlQueryModel* getAudittrail(QString sql,int order);
bool topUserUnlock(QString pwd);
int getTaskDeduct(QString id);
void writeLog(uchar type,QString msg);
private:
DatabaseThread();
static DatabaseThread *s_DatabaseThread;
QMutex mutex;
QSqlDatabase db;
};

#endif // DATABASETHREAD_H

void initDatabase()
{
	QsqlQuery qry(db);
	Qry.exec(“
/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET NAMES utf8 */;
/*!50503 SET NAMES utf8mb4 */;
/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
/*!40103 SET TIME_ZONE='+00:00' */;
/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;

-- 导出  表 trxdata_t.abcall 结构
CREATE TABLE IF NOT EXISTS `abcall` (
  `taskid` varchar(13) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL DEFAULT '',
  `sep` int unsigned NOT NULL DEFAULT (0),
  `sep_l` int unsigned NOT NULL DEFAULT (0),
  `dvalue` int unsigned NOT NULL DEFAULT (0),
  `xab` float NOT NULL DEFAULT (0),
  `xba` float NOT NULL DEFAULT (0),
  `eff1` float NOT NULL DEFAULT (0),
  `eff2` float NOT NULL DEFAULT (0),
  `bg1` float NOT NULL DEFAULT (0),
  `bg2` float NOT NULL DEFAULT (0),
  `bg3` float NOT NULL DEFAULT '0',
  `na1` float NOT NULL DEFAULT '0',
  `na2` float NOT NULL DEFAULT '0',
  `na3` float NOT NULL DEFAULT '0',
  `a1` float NOT NULL DEFAULT '0',
  `a2` float NOT NULL DEFAULT '0',
  `nb1` float NOT NULL DEFAULT '0',
  `nb2` float NOT NULL DEFAULT '0',
  `nb3` float NOT NULL DEFAULT '0',
  PRIMARY KEY (`taskid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.abcall_id 结构
CREATE TABLE IF NOT EXISTS `abcall_id` (
  `taskid` varchar(13) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL DEFAULT '',
  `samID` varchar(10) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL,
  `sep` int unsigned NOT NULL DEFAULT '0',
  `sep_l` int unsigned NOT NULL DEFAULT '0',
  `dvalue` int unsigned NOT NULL DEFAULT '0',
  `xab` float NOT NULL DEFAULT '0',
  `xba` float NOT NULL DEFAULT '0',
  `eff1` float NOT NULL DEFAULT '0',
  `eff2` float NOT NULL DEFAULT '0',
  `bg1` float NOT NULL DEFAULT '0',
  `bg2` float NOT NULL DEFAULT '0',
  `bg3` float NOT NULL DEFAULT '0',
  `na1` float NOT NULL DEFAULT '0',
  `na2` float NOT NULL DEFAULT '0',
  `na3` float NOT NULL DEFAULT '0',
  `a1` float NOT NULL DEFAULT '0',
  `a2` float NOT NULL DEFAULT '0',
  `nb1` float NOT NULL DEFAULT '0',
  `nb2` float NOT NULL DEFAULT '0',
  `nb3` float NOT NULL DEFAULT '0',
  PRIMARY KEY (`taskid`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.abhiscurve 结构
CREATE TABLE IF NOT EXISTS `abhiscurve` (
  `taskid` varchar(13) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  `sep` int unsigned NOT NULL DEFAULT '0',
  `xab` float NOT NULL DEFAULT (0),
  `xba` float NOT NULL DEFAULT (0),
  `chk` char(1) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL DEFAULT '1'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.abstandard 结构
CREATE TABLE IF NOT EXISTS `abstandard` (
  `taskid` varchar(13) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  `Nf` double NOT NULL DEFAULT (0),
  `Nft` double NOT NULL DEFAULT (0),
  `TDCR` float NOT NULL DEFAULT (0),
  `x_ab` float NOT NULL DEFAULT (0),
  `x_ba` float NOT NULL DEFAULT (0),
  `N1` double NOT NULL DEFAULT (0),
  `Eff1` float NOT NULL DEFAULT (0),
  `A1` double NOT NULL DEFAULT (0),
  `N2` double NOT NULL DEFAULT (0),
  `Eff2` float NOT NULL DEFAULT (0),
  `A2` double NOT NULL DEFAULT (0),
  `N3` double NOT NULL DEFAULT (0)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.abstandard_a 结构
CREATE TABLE IF NOT EXISTS `abstandard_a` (
  `samID` varchar(10) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL,
  `taskid` varchar(13) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  `Nf` double NOT NULL DEFAULT '0',
  `Nft` double NOT NULL DEFAULT '0',
  `TDCR` float NOT NULL DEFAULT '0',
  `x_ab` float NOT NULL DEFAULT '0',
  `x_ba` float NOT NULL DEFAULT '0',
  `N1` double NOT NULL DEFAULT '0',
  `Eff1` float NOT NULL DEFAULT '0',
  `A1` double NOT NULL DEFAULT '0',
  `N2` double NOT NULL DEFAULT '0',
  `N3` double NOT NULL DEFAULT '0'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.abstandard_b 结构
CREATE TABLE IF NOT EXISTS `abstandard_b` (
  `samID` varchar(10) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL,
  `taskid` varchar(13) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  `Nf` double NOT NULL DEFAULT '0',
  `Nft` double NOT NULL DEFAULT '0',
  `TDCR` float NOT NULL DEFAULT '0',
  `x_ab` float NOT NULL DEFAULT '0',
  `x_ba` float NOT NULL DEFAULT '0',
  `N1` double NOT NULL DEFAULT '0',
  `N2` double NOT NULL DEFAULT '0',
  `Eff2` float NOT NULL DEFAULT '0',
  `A2` double NOT NULL DEFAULT '0',
  `N3` double NOT NULL DEFAULT '0'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.abstandard_bg 结构
CREATE TABLE IF NOT EXISTS `abstandard_bg` (
  `samID` varchar(10) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL,
  `taskid` varchar(13) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  `Nf` double NOT NULL DEFAULT '0',
  `Nft` double NOT NULL DEFAULT '0',
  `TDCR` float NOT NULL DEFAULT '0',
  `N1` double NOT NULL DEFAULT '0',
  `N2` double NOT NULL DEFAULT '0',
  `N3` double NOT NULL DEFAULT '0'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.abstandard_id 结构
CREATE TABLE IF NOT EXISTS `abstandard_id` (
  `samID` varchar(10) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL,
  `taskid` varchar(13) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  `Nf` double NOT NULL DEFAULT '0',
  `Nft` double NOT NULL DEFAULT '0',
  `TDCR` float NOT NULL DEFAULT '0',
  `x_ab` float NOT NULL DEFAULT '0',
  `x_ba` float NOT NULL DEFAULT '0',
  `N1` double NOT NULL DEFAULT '0',
  `Eff1` float NOT NULL DEFAULT '0',
  `A1` double NOT NULL DEFAULT '0',
  `N2` double NOT NULL DEFAULT '0',
  `Eff2` float NOT NULL DEFAULT '0',
  `A2` double NOT NULL DEFAULT '0',
  `N3` double NOT NULL DEFAULT '0'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.alphabeta 结构
CREATE TABLE IF NOT EXISTS `alphabeta` (
  `name` varchar(50) NOT NULL DEFAULT '',
  `alpha` varchar(10) NOT NULL DEFAULT '',
  `active_a` double NOT NULL DEFAULT (0),
  `unit_a` char(1) NOT NULL DEFAULT '0',
  `date_a` date NOT NULL,
  `beta` varchar(10) NOT NULL DEFAULT '0',
  `active_b` double NOT NULL DEFAULT (0),
  `unit_b` char(1) NOT NULL DEFAULT '0',
  `date_b` date NOT NULL,
  `sep_s` int unsigned NOT NULL DEFAULT (22),
  `sep_l` int unsigned NOT NULL DEFAULT (80),
  `bz` tinytext,
  PRIMARY KEY (`name`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.bgdata 结构
CREATE TABLE IF NOT EXISTS `bgdata` (
  `taskid` varchar(30) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  `bz` tinytext CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci,
  `dt` datetime DEFAULT NULL,
  `times` int DEFAULT NULL,
  `pos` tinytext CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci,
  `Nd` double NOT NULL DEFAULT '0',
  `Nt` double NOT NULL DEFAULT '0',
  `Ns` double NOT NULL DEFAULT '0',
  `Nc` double NOT NULL DEFAULT '0',
  `Nct` double NOT NULL DEFAULT '0',
  `Nr` double NOT NULL DEFAULT '0',
  `Ndr` double NOT NULL DEFAULT '0',
  `ln1` double NOT NULL DEFAULT '0',
  `ln2` double NOT NULL DEFAULT '0',
  `ln3` double NOT NULL DEFAULT '0',
  PRIMARY KEY (`taskid`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.cpmdata 结构
CREATE TABLE IF NOT EXISTS `cpmdata` (
  `taskid` varchar(13) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  `Nf` double NOT NULL DEFAULT (0),
  `Nft` double NOT NULL DEFAULT (0),
  `TDCR` float NOT NULL DEFAULT (0),
  `N1` double NOT NULL DEFAULT (0),
  `N2` double NOT NULL DEFAULT (0),
  `N3` double NOT NULL DEFAULT (0)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.cpmdata_id 结构
CREATE TABLE IF NOT EXISTS `cpmdata_id` (
  `samID` varchar(10) DEFAULT NULL,
  `taskid` varchar(13) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  `Nf` double NOT NULL DEFAULT '0',
  `Nft` double NOT NULL DEFAULT '0',
  `TDCR` float NOT NULL DEFAULT '0',
  `N1` double NOT NULL DEFAULT '0',
  `N2` double NOT NULL DEFAULT '0',
  `N3` double NOT NULL DEFAULT '0'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.datarecord 结构
CREATE TABLE IF NOT EXISTS `datarecord` (
  `taskid` varchar(13) NOT NULL,
  `dt` datetime DEFAULT NULL,
  `times` decimal(8,3) DEFAULT NULL,
  `pos` tinytext,
  `samID` varchar(10) DEFAULT NULL,
  `SIE` float NOT NULL DEFAULT (0),
  `eTDCR` float NOT NULL DEFAULT (0),
  `Nd` double NOT NULL DEFAULT (0),
  `Nt` double NOT NULL DEFAULT (0),
  `Ns` double NOT NULL DEFAULT (0),
  `Nc` double NOT NULL DEFAULT (0),
  `Nct` double NOT NULL DEFAULT '0',
  `Nr` double NOT NULL DEFAULT (0),
  `Ndr` double NOT NULL DEFAULT (0),
  `ln1` double NOT NULL DEFAULT (0),
  `ln2` double NOT NULL DEFAULT (0),
  `ln3` double NOT NULL DEFAULT (0)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.dpmdata 结构
CREATE TABLE IF NOT EXISTS `dpmdata` (
  `taskid` varchar(13) NOT NULL,
  `Nf` double NOT NULL DEFAULT (0),
  `Nft` double NOT NULL DEFAULT (0),
  `TDCR` float NOT NULL DEFAULT (0),
  `N1` double NOT NULL DEFAULT (0),
  `N2` double NOT NULL DEFAULT (0),
  `N3` double NOT NULL DEFAULT (0),
  `Eff1` float NOT NULL DEFAULT (0),
  `A1` double NOT NULL DEFAULT (0),
  `LD` float NOT NULL DEFAULT (0),
  `FM` float DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.dpmdata_bg 结构
CREATE TABLE IF NOT EXISTS `dpmdata_bg` (
  `samID` varchar(10) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL,
  `taskid` varchar(13) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  `Nf` double NOT NULL DEFAULT '0',
  `Nft` double NOT NULL DEFAULT '0',
  `TDCR` float NOT NULL DEFAULT '0',
  `N1` double NOT NULL DEFAULT '0',
  `N2` double NOT NULL DEFAULT '0',
  `N3` double NOT NULL DEFAULT '0',
  `Eff1` float NOT NULL DEFAULT '0',
  `A1` double NOT NULL DEFAULT '0',
  `LD` float NOT NULL DEFAULT '0',
  `FM` float DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.dpmdata_id 结构
CREATE TABLE IF NOT EXISTS `dpmdata_id` (
  `samID` varchar(10) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL,
  `taskid` varchar(13) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  `Nf` double NOT NULL DEFAULT '0',
  `Nft` double NOT NULL DEFAULT '0',
  `TDCR` float NOT NULL DEFAULT '0',
  `N1` double NOT NULL DEFAULT '0',
  `N2` double NOT NULL DEFAULT '0',
  `N3` double NOT NULL DEFAULT '0',
  `Eff1` float NOT NULL DEFAULT '0',
  `A1` double NOT NULL DEFAULT '0',
  `LD` float NOT NULL DEFAULT '0',
  `FM` float DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.dpmdata_sa 结构
CREATE TABLE IF NOT EXISTS `dpmdata_sa` (
  `samID` varchar(10) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL,
  `taskid` varchar(13) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  `Nf` double NOT NULL DEFAULT '0',
  `Nft` double NOT NULL DEFAULT '0',
  `TDCR` float NOT NULL DEFAULT '0',
  `N1` double NOT NULL DEFAULT '0',
  `N2` double NOT NULL DEFAULT '0',
  `N3` double NOT NULL DEFAULT '0',
  `Eff1` float NOT NULL DEFAULT '0',
  `A1` double NOT NULL DEFAULT '0',
  `LD` float NOT NULL DEFAULT '0',
  `FM` float DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.errorinfo 结构
CREATE TABLE IF NOT EXISTS `errorinfo` (
  `dt` datetime NOT NULL,
  `info` tinytext NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.gbbackground 结构
CREATE TABLE IF NOT EXISTS `gbbackground` (
  `taskid` varchar(13) NOT NULL,
  `Nf` double NOT NULL DEFAULT (0),
  `Nft` double NOT NULL DEFAULT (0),
  `TDCR` float NOT NULL DEFAULT (0),
  `N1` double NOT NULL DEFAULT (0),
  `Eff1` float NOT NULL DEFAULT (0),
  `A1` double NOT NULL DEFAULT (0),
  `FM` float NOT NULL DEFAULT (0),
  `X2` float NOT NULL DEFAULT (0),
  `STD` float NOT NULL DEFAULT (0),
  `H3` char(1) NOT NULL DEFAULT ''
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.logs 结构
CREATE TABLE IF NOT EXISTS `logs` (
  `logid` int NOT NULL AUTO_INCREMENT,
  `dt` datetime NOT NULL,
  `user` varchar(20) NOT NULL,
  `type` tinyint NOT NULL DEFAULT '0',
  `msg` tinytext NOT NULL,
  `oldvalue` tinytext,
  `newvalue` tinytext,
  PRIMARY KEY (`logid`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=812 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.matchtopt 结构
CREATE TABLE IF NOT EXISTS `matchtopt` (
  `ptid` varchar(50) NOT NULL DEFAULT '',
  `bd` char(1) NOT NULL DEFAULT '0',
  `samID` varchar(10) DEFAULT NULL,
  PRIMARY KEY (`ptid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.noquench 结构
CREATE TABLE IF NOT EXISTS `noquench` (
  `name` varchar(50) NOT NULL,
  `H3` double NOT NULL DEFAULT (0),
  `H3unit` char(1) NOT NULL DEFAULT '0',
  `H3date` date NOT NULL,
  `C14` double NOT NULL DEFAULT (0),
  `C14unit` char(1) NOT NULL DEFAULT '0',
  `C14date` date NOT NULL,
  `bz` tinytext CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci,
  PRIMARY KEY (`name`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.normalization 结构
CREATE TABLE IF NOT EXISTS `normalization` (
  `cdate` date NOT NULL,
  `hv1` int NOT NULL DEFAULT (0),
  `hv2` int NOT NULL DEFAULT (0),
  `hv3` int NOT NULL DEFAULT '0'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.normalmeasure 结构
CREATE TABLE IF NOT EXISTS `normalmeasure` (
  `taskid` varchar(13) NOT NULL,
  `Nf` double NOT NULL DEFAULT (0),
  `Nft` double NOT NULL DEFAULT (0),
  `tdcr` float NOT NULL DEFAULT (0),
  `sis` float NOT NULL DEFAULT (0)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.nuclide 结构
CREATE TABLE IF NOT EXISTS `nuclide` (
  `name` varchar(20) NOT NULL,
  `halflife` float NOT NULL DEFAULT (0),
  `unit` char(1) NOT NULL DEFAULT '0',
  `type` char(1) NOT NULL DEFAULT '0',
  `energy` float NOT NULL DEFAULT (0),
  `hv1` int NOT NULL DEFAULT (0),
  `hv2` int NOT NULL DEFAULT (0),
  `hv3` int NOT NULL DEFAULT (0),
  `vts1` int NOT NULL DEFAULT (0),
  `vts2` int NOT NULL DEFAULT (0),
  `vts3` int NOT NULL DEFAULT (0),
  `mag` char(1) NOT NULL DEFAULT '0',
  `tcr` int NOT NULL DEFAULT (0),
  `tcr_` int NOT NULL DEFAULT (0),
  PRIMARY KEY (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.quench 结构
CREATE TABLE IF NOT EXISTS `quench` (
  `name` varchar(30) NOT NULL,
  `nuclide` varchar(20) NOT NULL,
  `num` int NOT NULL DEFAULT (0),
  `activity` double NOT NULL DEFAULT (0),
  `unit` char(1) NOT NULL DEFAULT '0',
  `calibrateddate` date NOT NULL,
  `bz` tinytext CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci,
  PRIMARY KEY (`name`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.quenchcurve 结构
CREATE TABLE IF NOT EXISTS `quenchcurve` (
  `taskid` varchar(13) NOT NULL,
  `Nf` double NOT NULL DEFAULT (0),
  `Nft` double NOT NULL DEFAULT (0),
  `TDCR` float NOT NULL DEFAULT (0),
  `eTDCR` float NOT NULL DEFAULT '0',
  `N1` double NOT NULL DEFAULT (0),
  `A1` double NOT NULL DEFAULT (0),
  `eff` float NOT NULL DEFAULT (0),
  `sie` float NOT NULL DEFAULT (0),
  `chk` char(1) NOT NULL DEFAULT '0'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.samdescribe 结构
CREATE TABLE IF NOT EXISTS `samdescribe` (
  `taskname` varchar(30) NOT NULL,
  `sup` int unsigned NOT NULL DEFAULT (0),
  `num` int unsigned NOT NULL DEFAULT (0),
  `describe` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT '0',
  `samID` varchar(10) DEFAULT NULL,
  PRIMARY KEY (`taskname`,`sup`,`num`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.specimen 结构
CREATE TABLE IF NOT EXISTS `specimen` (
  `name` varchar(20) NOT NULL,
  `nuclide` varchar(20) NOT NULL,
  `decaytype` char(1) NOT NULL DEFAULT '',
  `activity` double NOT NULL DEFAULT (0),
  `unit` char(1) NOT NULL DEFAULT '0',
  `calibrateddate` date NOT NULL,
  `bz` tinytext CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci,
  PRIMARY KEY (`name`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.task 结构
CREATE TABLE IF NOT EXISTS `task` (
  `name` varchar(30) NOT NULL,
  `menber` tinytext CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci,
  `type` int NOT NULL DEFAULT (0),
  `mode` int NOT NULL DEFAULT (0),
  `nuclide` varchar(10) NOT NULL DEFAULT '0',
  `time_jz` int NOT NULL DEFAULT (0),
  `time_zs` int NOT NULL DEFAULT (0),
  `time_cl` int NOT NULL DEFAULT (0),
  `jz_unit` char(1) NOT NULL DEFAULT '0',
  `zs_unit` char(1) NOT NULL DEFAULT '0',
  `cl_unit` char(1) NOT NULL DEFAULT '0',
  `repeat` int NOT NULL DEFAULT (0),
  `loop` int NOT NULL DEFAULT (0),
  `deduct` int NOT NULL DEFAULT (0),
  `dedv1` decimal(6,2) NOT NULL DEFAULT (0),
  `dedv2` decimal(6,2) NOT NULL DEFAULT (0),
  `dedv3` decimal(6,2) NOT NULL DEFAULT (0),
  `energy_unit` char(1) NOT NULL DEFAULT '0',
  `min1` int NOT NULL DEFAULT (0),
  `max1` int NOT NULL DEFAULT (0),
  `min2` int NOT NULL DEFAULT (0),
  `max2` int NOT NULL DEFAULT (0),
  `min3` int NOT NULL DEFAULT (0),
  `max3` int NOT NULL DEFAULT (0),
  `eff_refer` int NOT NULL DEFAULT (0),
  `eff1` varchar(30) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT '0',
  `eff2` varchar(30) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT '0',
  `eff3` varchar(30) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT '0',
  `esd` char(1) NOT NULL DEFAULT '0',
  `volume` decimal(4,2) NOT NULL DEFAULT (0),
  PRIMARY KEY (`name`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.tasklist 结构
CREATE TABLE IF NOT EXISTS `tasklist` (
  `name` varchar(30) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  `taskid` varchar(13) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  `dt` datetime NOT NULL,
  `complate` char(1) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL DEFAULT '0',
  `menber` tinytext CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci,
  `type` int NOT NULL DEFAULT '0',
  `mode` int NOT NULL DEFAULT '0',
  `nuclide` varchar(10) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL DEFAULT '0',
  `time_jz` int NOT NULL DEFAULT '0',
  `time_zs` int NOT NULL DEFAULT '0',
  `time_cl` int NOT NULL DEFAULT '0',
  `jz_unit` char(1) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL DEFAULT '0',
  `zs_unit` char(1) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL DEFAULT '0',
  `cl_unit` char(1) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL DEFAULT '0',
  `repeat` int NOT NULL DEFAULT '0',
  `loop` int NOT NULL DEFAULT '0',
  `deduct` int NOT NULL DEFAULT '0',
  `dedv1` decimal(6,2) NOT NULL DEFAULT '0.00',
  `dedv2` decimal(6,2) NOT NULL DEFAULT '0.00',
  `dedv3` decimal(6,2) NOT NULL DEFAULT '0.00',
  `energy_unit` char(1) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL DEFAULT '0',
  `min1` int NOT NULL DEFAULT '0',
  `max1` int NOT NULL DEFAULT '0',
  `min2` int NOT NULL DEFAULT '0',
  `max2` int NOT NULL DEFAULT '0',
  `min3` int NOT NULL DEFAULT '0',
  `max3` int NOT NULL DEFAULT '0',
  `eff_refer` int NOT NULL DEFAULT '0',
  `eff1` varchar(30) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT '0',
  `eff2` varchar(30) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT '0',
  `eff3` varchar(30) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT '0',
  `esd` char(1) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL DEFAULT '0',
  `volume` decimal(4,2) NOT NULL DEFAULT '0.00',
  `operator` tinytext NOT NULL,
  PRIMARY KEY (`taskid`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.taskrange 结构
CREATE TABLE IF NOT EXISTS `taskrange` (
  `taskid` varchar(13) NOT NULL,
  `samID` varchar(10) NOT NULL,
  `min1` int NOT NULL DEFAULT (0),
  `max1` int NOT NULL DEFAULT (0),
  `min2` int NOT NULL DEFAULT (0),
  `max2` int NOT NULL DEFAULT (0),
  `min3` int NOT NULL DEFAULT (0),
  `max3` int NOT NULL DEFAULT (0)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.tasksam 结构
CREATE TABLE IF NOT EXISTS `tasksam` (
  `taskid` varchar(13) NOT NULL,
  `sup` int unsigned NOT NULL DEFAULT (0),
  `num` int unsigned NOT NULL DEFAULT (0),
  `describe` varchar(20) DEFAULT NULL,
  `samID` varchar(10) DEFAULT NULL,
  PRIMARY KEY (`taskid`,`sup`,`num`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.test_task 结构
CREATE TABLE IF NOT EXISTS `test_task` (
  `taskid` varchar(30) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  `addr` int NOT NULL,
  `count` int DEFAULT '0',
  PRIMARY KEY (`taskid`,`addr`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  表 trxdata_t.trxuser 结构
CREATE TABLE IF NOT EXISTS `trxuser` (
  `uname` varchar(20) NOT NULL,
  `pwd` varchar(20) NOT NULL,
  `authority` char(1) NOT NULL DEFAULT '',
  `state` char(1) NOT NULL DEFAULT '0' COMMENT '0首次1正常2锁定3删除',
  `validity` date DEFAULT NULL,
  PRIMARY KEY (`uname`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

-- 数据导出被取消选择。

-- 导出  触发器 trxdata_t.add_alphabeta 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `add_alphabeta` AFTER INSERT ON `alphabeta` FOR EACH ROW BEGIN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`newvalue`) VALUES (NOW(),@cur_user,18,'新增ab库',
	CONCAT(NEW.`name`,',',NEW.`alpha`,',',NEW.`active_a`,',',NEW.`unit_a`,',',NEW.`date_a`,',',NEW.`beta`,',',NEW.`active_b`,',',NEW.`unit_b`,',',NEW.`date_b`,',',
	NEW.`sep_s`,',',NEW.`sep_l`));
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.add_bgdata 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `add_bgdata` AFTER INSERT ON `bgdata` FOR EACH ROW BEGIN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`newvalue`) VALUES (NOW(),@cur_user,24,'新增本底',
	CONCAT(NEW.`taskid`,',',NEW.`bz`,',',NEW.`times`,',',NEW.`pos`,',',NEW.`Nd`,',',NEW.`Nt`,',',NEW.`Ns`,',',NEW.`Nc`,',',NEW.`Nct`,',',
	NEW.`Nr`,',',NEW.`Ndr`,',',NEW.`ln1`,',',NEW.`ln2`,',',NEW.`ln3`));
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.add_noquench 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `add_noquench` AFTER INSERT ON `noquench` FOR EACH ROW BEGIN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`newvalue`) VALUES (NOW(),@cur_user,9,'新增无淬灭库',
	CONCAT(NEW.`name`,',',NEW.H3,',',NEW.`H3unit`,',',NEW.H3date,',',NEW.C14,',',NEW.C14unit,',',NEW.C14date));
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.add_nuclide 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `add_nuclide` AFTER INSERT ON `nuclide` FOR EACH ROW BEGIN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`newvalue`) VALUES (NOW(),@cur_user,6,'新增核素库',
	CONCAT(NEW.`name`,',',NEW.halflife,',',NEW.`unit`,',',NEW.`type`,',',NEW.`energy`,',',NEW.hv1,',',NEW.hv2,',',
	NEW.hv3,',',NEW.vts1,',',NEW.vts2,',',NEW.vts3,',',NEW.mag,',',NEW.tcr,',',NEW.tcr_));
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.add_quench 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `add_quench` AFTER INSERT ON `quench` FOR EACH ROW BEGIN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`newvalue`) VALUES (NOW(),@cur_user,12,'新增系列淬灭库',
	CONCAT(NEW.`name`,',',NEW.`nuclide`,',',NEW.`num`,',',NEW.activity,',',NEW.unit,',',NEW.calibrateddate));
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.add_specimen 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `add_specimen` AFTER INSERT ON `specimen` FOR EACH ROW BEGIN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`newvalue`) VALUES (NOW(),@cur_user,15,'新增标样库',
	CONCAT(NEW.`name`,',',NEW.`nuclide`,',',NEW.`decaytype`,',',NEW.activity,',',NEW.unit,',',NEW.calibrateddate));

END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.add_task 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `add_task` AFTER INSERT ON `task` FOR EACH ROW BEGIN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`newvalue`) VALUES (NOW(),@cur_user,1,'新增任务',CONCAT(NEW.`name`,',',NEW.`menber`,',',NEW.`type`,',',NEW.`mode`,',',
	NEW.`nuclide`,',',NEW.time_jz,',',NEW.time_zs,',',NEW.time_cl,',',NEW.jz_unit,',',NEW.zs_unit,',',NEW.cl_unit,',',NEW.`repeat`,',',NEW.`loop`,',',NEW.deduct,',',
	NEW.dedv1,',',NEW.dedv2,',',NEW.dedv3,',',NEW.energy_unit,',',NEW.min1,',',NEW.max1,',',NEW.min2,',',NEW.max2,',',NEW.min3,',',
NEW.max3,',',NEW.eff_refer,',',IFNULL(NEW.eff1,'-'),',',IFNULL(NEW.eff2,'-'),',',IFNULL(NEW.eff3,'-'),',',NEW.esd) );
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.add_tasklist 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `add_tasklist` AFTER INSERT ON `tasklist` FOR EACH ROW BEGIN
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`newvalue`) VALUES (NOW(),@cur_user,4,'开始任务',
	CONCAT(NEW.`name`,',',NEW.`taskid`,',',NEW.`menber`,',',NEW.`type`,',',NEW.`mode`,',',
	NEW.`nuclide`,',',NEW.time_jz,',',NEW.time_zs,',',NEW.time_cl,',',NEW.jz_unit,',',NEW.zs_unit,',',NEW.cl_unit,',',NEW.`repeat`,',',NEW.`loop`,',',NEW.deduct,',',
	NEW.dedv1,',',NEW.dedv2,',',NEW.dedv3,',',NEW.min1,',',NEW.max1,',',NEW.min2,',',NEW.max2,',',NEW.min3,',',
NEW.max3,',',NEW.eff_refer,',',IFNULL(NEW.eff1,'-'),',',IFNULL(NEW.eff2,'-'),',',IFNULL(NEW.eff3,'-'),',',NEW.esd) );
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.del_alphabeta 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `del_alphabeta` AFTER DELETE ON `alphabeta` FOR EACH ROW BEGIN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`oldvalue`) VALUES (NOW(),@cur_user,20,'删除ab库',
	CONCAT(OLD.`name`,',',OLD.`alpha`,',',OLD.`active_a`,',',OLD.`unit_a`,',',OLD.`date_a`,',',OLD.`beta`,',',OLD.`active_b`,',',OLD.`unit_b`,',',OLD.`date_b`,',',OLD.`sep_s`,',',OLD.`sep_l`));

END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.del_bgdata 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `del_bgdata` AFTER DELETE ON `bgdata` FOR EACH ROW BEGIN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`oldvalue`) VALUES (NOW(),@cur_user,25,'删除本底',
	CONCAT(OLD.`taskid`,',',OLD.`bz`,',',OLD.`times`,',',OLD.`pos`,',',OLD.`Nd`,',',OLD.`Nt`,',',OLD.`Ns`,',',OLD.`Nc`,',',OLD.`Nct`,',',
	OLD.`Nr`,',',OLD.`Ndr`,',',OLD.`ln1`,',',OLD.`ln2`,',',OLD.`ln3`));
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.del_noquench 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `del_noquench` AFTER DELETE ON `noquench` FOR EACH ROW BEGIN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`oldvalue`) VALUES (NOW(),@cur_user,11,'删除无淬灭库',
	CONCAT(OLD.`name`,',',OLD.H3,',',OLD.`H3unit`,',',OLD.H3date,',',OLD.C14,',',OLD.C14unit,',',OLD.C14date));
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.del_normalization 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `del_normalization` AFTER DELETE ON `normalization` FOR EACH ROW BEGIN
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`oldvalue`) VALUES (NOW(),@cur_user,27,'删除历史检验数据',CONCAT(OLD.cdate,',',OLD.hv1,',',OLD.hv2,',',OLD.hv3));
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.del_nuclide 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `del_nuclide` AFTER DELETE ON `nuclide` FOR EACH ROW BEGIN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`oldvalue`) VALUES (NOW(),@cur_user,8,'删除核素库',
	CONCAT(OLD.`name`,',',OLD.halflife,',',OLD.`unit`,',',OLD.`type`,',',OLD.`energy`,',',OLD.hv1,',',OLD.hv2,',',
	OLD.hv3,',',OLD.vts1,',',OLD.vts2,',',OLD.vts3,',',OLD.mag,',',OLD.tcr,',',OLD.tcr_));
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.del_quench 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `del_quench` AFTER DELETE ON `quench` FOR EACH ROW BEGIN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`oldvalue`) VALUES (NOW(),@cur_user,14,'删除系列淬灭库',
	CONCAT(OLD.`name`,',',OLD.`nuclide`,',',OLD.`num`,',',OLD.activity,',',OLD.unit,',',OLD.calibrateddate));
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.del_specimen 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `del_specimen` AFTER DELETE ON `specimen` FOR EACH ROW BEGIN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`oldvalue`) VALUES (NOW(),@cur_user,17,'删除标样库',
	CONCAT(OLD.`name`,',',OLD.`nuclide`,',',OLD.`decaytype`,',',OLD.activity,',',OLD.unit,',',OLD.calibrateddate));
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.del_task 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `del_task` AFTER DELETE ON `task` FOR EACH ROW BEGIN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`oldvalue`) VALUES (NOW(),@cur_user,3,'删除任务',
	CONCAT(OLD.`name`,',',OLD.`menber`,',',OLD.`type`,',',OLD.`mode`,',',OLD.`nuclide`,',',OLD.time_jz,',',OLD.time_zs,',',OLD.time_cl,',',OLD.jz_unit,',',OLD.zs_unit,',',OLD.cl_unit,
',',OLD.`repeat`,',',OLD.`loop`,',',OLD.deduct,',',OLD.dedv1,',',OLD.dedv2,',',OLD.dedv3,',',OLD.energy_unit,',',OLD.min1,',',OLD.max1,',',OLD.min2,',',OLD.max2,',',OLD.min3,',',
OLD.max3,',',OLD.eff_refer,',',IFNULL(OLD.eff1,'-'),',',IFNULL(OLD.eff2,'-'),',',IFNULL(OLD.eff3,'-'),',',OLD.esd));
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.del_tasklist 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `del_tasklist` AFTER DELETE ON `tasklist` FOR EACH ROW BEGIN
	IF OLD.complate=0 THEN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`oldvalue`) VALUES (NOW(),'system',21,'自动删除无数据未完成任务',CONCAT(OLD.`name`,',',OLD.`taskid`));
	ELSE 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`oldvalue`) VALUES (NOW(),@cur_user,21,'删除历史任务',CONCAT(OLD.`name`,',',OLD.`taskid`));
	END IF;
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.up_alphabeta 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `up_alphabeta` BEFORE UPDATE ON `alphabeta` FOR EACH ROW BEGIN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`oldvalue`,`newvalue`) VALUES (NOW(),@cur_user,19,'更新ab库', 
CONCAT(OLD.`name`,',',OLD.`alpha`,',',OLD.`active_a`,',',OLD.`unit_a`,',',OLD.`date_a`,',',OLD.`beta`,',',OLD.`active_b`,',',OLD.`unit_b`,',',OLD.`date_b`,',',OLD.`sep_s`,',',OLD.`sep_l`),
CONCAT(NEW.`name`,',',NEW.`alpha`,',',NEW.`active_a`,',',NEW.`unit_a`,',',NEW.`date_a`,',',NEW.`beta`,',',NEW.`active_b`,',',NEW.`unit_b`,',',NEW.`date_b`,',',NEW.`sep_s`,',',NEW.`sep_l`));
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.up_bgdata 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `up_bgdata` BEFORE UPDATE ON `bgdata` FOR EACH ROW BEGIN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`oldvalue`,`newvalue`) VALUES (NOW(),@cur_user,26,'更新本底备注',OLD.bz,NEW.bz );
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.up_noquench 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `up_noquench` BEFORE UPDATE ON `noquench` FOR EACH ROW BEGIN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`oldvalue`,`newvalue`) VALUES (NOW(),@cur_user,10,'更新无淬灭库', 
CONCAT(OLD.`name`,',',OLD.H3,',',OLD.`H3unit`,',',OLD.H3date,',',OLD.C14,',',OLD.C14unit,',',OLD.C14date),
CONCAT(NEW.`name`,',',NEW.H3,',',NEW.`H3unit`,',',NEW.H3date,',',NEW.C14,',',NEW.C14unit,',',NEW.C14date));
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.up_nuclide 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `up_nuclide` BEFORE UPDATE ON `nuclide` FOR EACH ROW BEGIN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`oldvalue`,`newvalue`) VALUES (NOW(),@cur_user,7,'更新核素库', 
CONCAT(OLD.`name`,',',OLD.halflife,',',OLD.`unit`,',',OLD.`type`,',',OLD.`energy`,',',OLD.hv1,',',OLD.hv2,',',OLD.hv3,',',OLD.vts1,',',OLD.vts2,',',OLD.vts3,',',OLD.mag,',',OLD.tcr,',',OLD.tcr_),
CONCAT(NEW.`name`,',',NEW.halflife,',',NEW.`unit`,',',NEW.`type`,',',NEW.`energy`,',',NEW.hv1,',',NEW.hv2,',',NEW.hv3,',',NEW.vts1,',',NEW.vts2,',',NEW.vts3,',',NEW.mag,',',NEW.tcr,',',NEW.tcr_));
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.up_quench 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `up_quench` BEFORE UPDATE ON `quench` FOR EACH ROW BEGIN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`oldvalue`,`newvalue`) VALUES (NOW(),@cur_user,13,'更新系列淬灭库', 
CONCAT(OLD.`name`,',',OLD.`nuclide`,',',OLD.`num`,',',OLD.activity,',',OLD.unit,',',OLD.calibrateddate),
CONCAT(NEW.`name`,',',NEW.`nuclide`,',',NEW.`num`,',',NEW.activity,',',NEW.unit,',',NEW.calibrateddate));
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.up_specimen 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `up_specimen` BEFORE UPDATE ON `specimen` FOR EACH ROW BEGIN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`oldvalue`,`newvalue`) VALUES (NOW(),@cur_user,16,'更新标样库', 
CONCAT(OLD.`name`,',',OLD.`nuclide`,',',OLD.`decaytype`,',',OLD.activity,',',OLD.unit,',',OLD.calibrateddate),
CONCAT(NEW.`name`,',',NEW.`nuclide`,',',NEW.`decaytype`,',',NEW.activity,',',NEW.unit,',',NEW.calibrateddate));
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.up_task 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `up_task` BEFORE UPDATE ON `task` FOR EACH ROW BEGIN IF OLD.energy_unit=NEW.energy_unit THEN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`oldvalue`,`newvalue`) VALUES (NOW(),@cur_user,2,'更新任务', 
CONCAT(OLD.`name`,',',OLD.`menber`,',',OLD.`type`,',',OLD.`mode`,',',OLD.`nuclide`,',',OLD.time_jz,',',OLD.time_zs,',',OLD.time_cl,',',OLD.jz_unit,',',OLD.zs_unit,',',OLD.cl_unit,
',',OLD.`repeat`,',',OLD.`loop`,',',OLD.deduct,',',OLD.dedv1,',',OLD.dedv2,',',OLD.dedv3,',',OLD.energy_unit,',',OLD.min1,',',OLD.max1,',',OLD.min2,',',OLD.max2,',',OLD.min3,',',
OLD.max3,',',OLD.eff_refer,',',IFNULL(OLD.eff1,'-'),',',IFNULL(OLD.eff2,'-'),',',IFNULL(OLD.eff3,'-'),',',OLD.esd),
CONCAT(NEW.`name`,',',NEW.`menber`,',',NEW.`type`,',',NEW.`mode`,',',NEW.`nuclide`,',',NEW.time_jz,',',NEW.time_zs,',',NEW.time_cl,',',NEW.jz_unit,',',NEW.zs_unit,',',NEW.cl_unit,
',',NEW.`repeat`,',',NEW.`loop`,',',NEW.deduct,',',NEW.dedv1,',',NEW.dedv2,',',NEW.dedv3,',',NEW.energy_unit,',',NEW.min1,',',NEW.max1,',',NEW.min2,',',NEW.max2,',',NEW.min3,',',
NEW.max3,',',NEW.eff_refer,',',IFNULL(NEW.eff1,'-'),',',IFNULL(NEW.eff2,'-'),',',IFNULL(NEW.eff3,'-'),',',NEW.esd));
END IF ;
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.up_tasklist 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `up_tasklist` AFTER UPDATE ON `tasklist` FOR EACH ROW BEGIN
	DECLARE flag INT DEFAULT 0;
	IF OLD.`complate` != NEW.`complate` THEN
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`) VALUES (NOW(),@cur_user,5,CONCAT('完成任务:',OLD.`name`,',',OLD.`taskid`) );
	ELSEIF OLD.eff1 != NEW.eff1 THEN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`oldvalue`,`newvalue`) VALUES (NOW(),@cur_user,41,CONCAT('更改调用曲线:',OLD.`name`,',',OLD.`taskid`),OLD.eff1,NEW.eff1);
	ELSE
	BEGIN
	IF OLD.min1!=NEW.min1 THEN SET flag=flag+1; END IF;
	IF OLD.max1!=NEW.max1 THEN SET flag=flag+2; END IF;
	IF OLD.min2!=NEW.min2 THEN SET flag=flag+4; END IF;
	IF OLD.max2!=NEW.max2 THEN SET flag=flag+8; END IF;
	IF OLD.min3!=NEW.min3 THEN SET flag=flag+16; END IF;
	IF OLD.max3!=NEW.max3 THEN SET flag=flag+32; END IF;
	IF flag>0 THEN 
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`oldvalue`,`newvalue`) VALUES (NOW(),@cur_user,40,CONCAT('更改窗口:',OLD.`name`,',',OLD.`taskid`),
	CONCAT(OLD.min1,',',OLD.max1,',',OLD.min2,',',OLD.max2,',',OLD.min3,',',OLD.max3),CONCAT(NEW.min1,',',NEW.max1,',',NEW.min2,',',NEW.max2,',',NEW.min3,',',NEW.max3));
	END IF;
	END ;
	END IF;
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;

-- 导出  触发器 trxdata_t.up_taskrange 结构
SET @OLDTMP_SQL_MODE=@@SQL_MODE, SQL_MODE='ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION';
DELIMITER //
CREATE TRIGGER `up_taskrange` AFTER UPDATE ON `taskrange` FOR EACH ROW BEGIN
	INSERT INTO `logs` (`dt`,`user`,`type`,`msg`,`oldvalue`,`newvalue`) VALUES (NOW(),@cur_user,30,CONCAT('更新ID窗口:',OLD.`taskid`,',',OLD.`samID`),
	CONCAT(OLD.min1,',',OLD.max1,',',OLD.min2,',',OLD.max2,',',OLD.min3,',',OLD.max3),CONCAT(NEW.min1,',',NEW.max1,',',NEW.min2,',',NEW.max2,',',NEW.min3,',',NEW.max3));  
END//
DELIMITER ;
SET SQL_MODE=@OLDTMP_SQL_MODE;
/*!40103 SET TIME_ZONE=IFNULL(@OLD_TIME_ZONE, 'system') */;
/*!40101 SET SQL_MODE=IFNULL(@OLD_SQL_MODE, '') */;
/*!40014 SET FOREIGN_KEY_CHECKS=IFNULL(@OLD_FOREIGN_KEY_CHECKS, 1) */;
/*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */;
/*!40111 SET SQL_NOTES=IFNULL(@OLD_SQL_NOTES, 1) */;
”);
}

int DatabaseThread::userLogin(QString user, QString pwd)
{
    QSqlQuery qry(db);
    qry.exec("SELECT `pwd`,`authority`,`state`,`validity` FROM `trxuser` WHERE `uname`='"+user+"'");
    if(qry.first()){
        QString dbpwd = qry.value(0).toString();
        int auth = qry.value(1).toInt();
        int state = qry.value(2).toInt();
        QDate dt = qry.value(3).toDate();
        if(dt.isValid() && dt<QDate::currentDate())
            state = 4;
        if(pwd==dbpwd){
            g_user = user;
            g_pwd = pwd;
            g_authority = auth;
            switch (state) {
            case 0:
                writeLog(0,"首次登录");
                break;
            case 1:
                writeLog(0,"登录成功");
                break;
            case 2:
                writeLog(0,"登录失败：账号已锁定");
                break;
            case 3:
                writeLog(0,"登录失败：账号已注销");
                break;
            }
            qry.exec("SET @cur_user = '"+user+"';");
            return state;
        }
        else{
            writeLog(0,"登录失败：密码错误");
            return -1;
        }
    }
    else{
        writeLog(0,"登录失败：用户名不存在");
        return -2;
    }
}

QStringList DatabaseThread::getUserList(bool containAdmin)
{
    QMutexLocker locker(&mutex);
    QSqlQuery qry(db);
    QStringList lst;
    if(containAdmin)
        qry.exec("SELECT `uname`,case `authority` when 0 then '管理' when 1 then '操作员' END AS 'type' FROM `trxuser`");
    else
        qry.exec("SELECT `uname`,case `authority` when 1 then '操作员' when 2 then '操作员' END AS 'type' FROM `trxuser` WHERE `authority`!=0");
    while (qry.next()) {
        lst.append(qry.value(0).toString()+"|"+qry.value(1).toString());
    }
    return lst;
}

void DatabaseThread::updateUser(QString uname,QString pwd, bool fzr, bool mode)
{
    QMutexLocker locker(&mutex);
    QSqlQuery qry(db);
    QString str;
    if(mode)
        str = "INSERT INTO `trxuser` (`pwd`,`authority`,`uname`) VALUES (?,?,?)";
    else
        str = "UPDATE `trxuser` SET `pwd`=?,`authority`=? WHERE `uname`=?";
    qry.prepare(str);
    int auth=2;
    if(fzr)
        auth=1;
    qry.bindValue(0,pwd);
    qry.bindValue(1,auth);
    qry.bindValue(2,uname);
    qry.exec();
}

void DatabaseThread::updateUser(QString uname, QString pwd)
{
    QMutexLocker locker(&mutex);
    QSqlQuery qry(db);
    qry.prepare("UPDATE `trxuser` SET `pwd`=?,`state`=1 WHERE `uname`=?");
    qry.bindValue(0,pwd);
    qry.bindValue(1,uname);
    qry.exec();
}

void DatabaseThread::deleteUser(QString uname)
{
    QMutexLocker locker(&mutex);
    QSqlQuery qry(db);
    qry.exec("DELETE FROM `trxuser` WHERE `uname` = '"+uname+"'");
}

void DatabaseThread::lockUser(QString uname)
{
    QSqlQuery qry(db);
    qry.exec("update `trxuser` set `state`=2 where `uname`='"+uname+"'");
}

QSqlQueryModel *DatabaseThread::getAudittrail(QString sql,int order)
{
    QMutexLocker locker(&mutex);
    audit_model->setQuery(sql,db);
    audit_model->setHeaderData(0,Qt::Horizontal,"时间");
    audit_model->setHeaderData(1,Qt::Horizontal,"用户名");
    audit_model->setHeaderData(2,Qt::Horizontal,"事件类型");
    audit_model->setHeaderData(3,Qt::Horizontal,"内容");
    audit_model->setHeaderData(4,Qt::Horizontal,"旧值");
    audit_model->setHeaderData(5,Qt::Horizontal,"新值");
    switch (order) {
    case -1:
        audit_model->setHeaderData(0,Qt::Horizontal,"时间↓");
        break;
    case 1:
        audit_model->setHeaderData(0,Qt::Horizontal,"时间↑");
        break;
    case -2:
        audit_model->setHeaderData(1,Qt::Horizontal,"用户名↓");
        break;
    case 2:
        audit_model->setHeaderData(1,Qt::Horizontal,"用户名↑");
        break;
    case -3:
        audit_model->setHeaderData(2,Qt::Horizontal,"事件类型↓");
        break;
    case 3:
        audit_model->setHeaderData(2,Qt::Horizontal,"事件类型↑");
        break;
    }
    qDebug()<<audit_model->rowCount();
    return audit_model;
}

bool DatabaseThread::topUserUnlock(QString pwd)
{
    QMutexLocker locker(&mutex);
    QSqlQuery qry(db);
    qry.exec("SELECT * FROM `trxuser` WHERE `authority`=0 AND `pwd`='"+pwd+"'");
    return qry.first();
}

void DatabaseThread::writeLog(uchar type, QString msg)
{
    QSqlQuery qry(db);
    qry.prepare("INSERT INTO `logs` (`dt`,`user`,`type`,`msg`) VALUES (NOW(),?,?,?)");
    if(g_user.isEmpty())
        qry.bindValue(0,"system");
    else
        qry.bindValue(0,g_user);
    qry.bindValue(1,type);
    qry.bindValue(2,msg);
    bool ok=qry.exec();
    if(!ok)
        qDebug()<<qry.lastError().text();
}
