# NIP-5135 Send daily COD return notification email

- [x] SpecNote writing...
This batch will send an email notification with daily COD return confirmation report file to `<LOD>`.	
'- The report file will be named: sagawa_cod_return_confirmation_report_YYYYMMDD.csv, includes
  1. MOVEX Invoice Number
  2. Sagawa Dispatch Number."
'- The records must complied with conditions below:
  1. Check i_in_return table for:
    a. d_dp, i_dpslot equals current c_dppara record.
    b. c_processing = Sk.cprocessing_WAIT
    c. c_redelivery = Sk.credelivery_RETURN
  2. Select from p_invsum by the i_invoice of i_in_return records above, which c_paymode = Sk.cpaymode_COD"
'- In order to send email notification on this occasion, some steps of setting needs to be done.
  1. Create new controller for the report mail.
  2. Create an email template.
  3. Create a new c_message record.
'- Setting the right stage in dprun.conf 

## DEV

```yml
# config/env_server_prod.yml
<EMAIL_COD_RETURN_NOTIFICATION> : "XXX@ottoxxxx.co.jp, mario@xxx.xxx.tw"
```

```
modules/warehouse/amos_warehouse_batches/app/batches/cod_return_list.rb
rails/amos/app/controller/cod_return_notification.rb
rails/amos/app/views/cod_return_notification/_cod_return_notification.erb

```

```ruby=

# rescue missing template
# rails/amos/app/views/cod_return_notification/send_mail.efb


```




---

```ruby=
rtcode = plsql.ifp_notify_nat.insert_message(
  n_company: 50,
  n_cust_company: 5002,
  v_msgcode: "MSG_COD_RETURN",
  v_parameters: "<record_count>3</record_count><no>1</no><invoiceno>946286476</invoiceno><i_parcel>153131829425</i_parcel><no>2</no><invoiceno>784077896</invoiceno><i_parcel>632038164548</i_parcel><no>3</no><invoiceno>717354021</invoiceno><i_parcel>717540670964</i_parcel>",
  n_account: 0,
  v_language: "en",
  v_msgmode: "I",
  v_opcode: "AMO",
  n_order: 0,
  v_errorcode: nil
)
```





## Backup codes

- Batch: CSV file export function
```ruby=
#+smile::script
class SagawaCodReturnCheckingReport < AmosBase::Batches::Base
  def initialize(*args)
    super(*args)
    @asc_path = GetConfig.get_entry('PATH', 'AscData')
    @timestamp = Time.current.strftime('%Y%m%d')
    @company = Sk.icompany_EBJ
    @cust_company = Sk.icustcompany_B2B
  end

  def asc_path
    @asc_path ||= GetConfig.get_entry('PATH', 'AscData')
  end

  def file_name
    @file_name ||= "sagawa_cod_return_checking_report_#{@timestamp}.csv"
  end

  def run_internal
  fail(ActiveRecord::RecordNotFound, 'No record in CDppara!') unless set_dpdate_and_slot

  records = IInReturn.includes(:invoice)
                      .where(d_dp: @dpdate,
                            i_dpslot: @dpslot,
                            c_processing: Sk.iinreturn_cprocessing_WAIT,
                            c_redelivery: Sk.credelivery_RETURN,
                            p_invsum: { c_paymode: Sk.cpaymode_COD, c_paid: Sk.invsum_cpaid_NO })
                      .order(i_invoice: :desc)

  records.present? ? insert_i_out_messages : log.info('No COD return records need to be report.')

  OK
rescue => exception
  log.error("error #{exception.message}")
  log.error("#{exception.backtrace.join("\n")}")
  inc_error_count
  PROGERROR
  end

  def set_dpdate_and_slot
    cdppara = CDppara.find_cached

    if cdppara
        @dpdate = cdppara.d_dp
        @dpslot = cdppara.i_dpslot
    else
      return false
    end

    true
  end

  def write_file(records)
    file = File.join(asc_path, file_name)

    CSV.open(file, "wb") do |csv|
      csv << ['No.', 'MOVEX Invoice Number', 'Sagawa Dispatch Number']

      records.each do |record|
        next if record.invoice.blank?
        csv << [inc_record_count, record.i_invoice, record.invoice.i_parcel]
      end
    end

    insert_i_out_messages
  end

  def insert_i_out_messages
    opcode = ENV['OPCODE']

    s_parameter = set_s_parameter(@record_count, file_name)
    c_msgmode = Sk.message_cmsgmode_IMMEDIATELY

    IOutMessage.insert_message(@company, @cust_company, 'MSG_COD_RETURN_CHECK',
                               s_parameter, nil, "en", c_msgmode, opcode)
  end

  def set_s_parameter(record_count, file_name)
    "<record_count>#{record_count}</record_count><file_name>#{file_name}</file_name>"
  end
end
```

- Controller: import file and add attachment
```ruby=
# Send an email notification with daily COD return checking report file to <LOD>.
require 'controller_helper'

class CodReturnNotification < ActionMailer::Base
  add_template_helper(AmosActionMailer::Helpers::NotificationsHelper)

  # ===Description
  # AmosActionMailer is executed by amos_base_send_message,
  # in which it always calls send_mail(ioutmessage, cmessage, pcustomer).deliver_now,
  # that's why it needs a redundant parameter pcustomer.
  def send_mail(ioutmessage, cmessage, pcustomer = nil)
    # parse params from i_out_message to set @record_count, @file_name 
    parse_ioutmessage(ioutmessage.params_as_hash)

    asc_path = GetConfig.get_entry('PATH', 'AscData')
    @file = File.join(asc_path, @file_name)
    attachments[params['file_name']] = File.read(@file)
    do_not_deliver! if @file.blank?

    if @record_count >= 5
      @report_in_mail = CSV.read(@file, headers: true, header_converters: :symbol, converters: :all)
    end

    # setup mail header
    @sent_on = Time.current
    @subject = 'Return list with unpaid COD invoices'
    @from = cmessage.s_from_address
    @to = cmessage.s_to_address
    @cc = cmessage.s_cc_address
    @bcc = cmessage.s_bcc_address
    @reply_to = cmessage.s_replyto_address
    @content_type = 'text/html'

    @ioutmessage = ioutmessage
    @template_partial_name = case ioutmessage.s_msgcode
                             when 'MSG_COD_RETURN_CHECK ' then 'cod_return_checking_report'
                             else 'send_mail'
                             end
    mail(from: @from, to: @to, cc: @cc, bcc: @bcc, reply_to: @reply_to, subject: @subject, content_type: 'text/html')
  end

  def parse_ioutmessage(params)
    @record_count = params['record_count']
    @file_name = params['file_name']
  end
end
```
- Batch test: test CSV file
```ruby
require File.dirname(__FILE__) + '/../test_helper'
#
class CodReturnListTest < AmosBase::Util::SimpleTestCase
  include AmosBase::Util::Test::ActiveRecordAssertions

  def setup
    super
    @asc_path ||= GetConfig.getEntry('PATH', 'AscData')
    @asc_path = GetConfig.get_entry('PATH', 'AscData')
    @paymode = Sk.cpaymode_COD
    @timestamp = Time.current.strftime('%Y%m%d')
    @file_name = "sagawa_cod_return_checking_report_#{@timestamp}.csv"
    @file = File.join(@asc_path, @file_name)
    @cdppara = CDppara.find_cached || CDppara.make
    @i_company = Sk.icompany_EBJ
    @i_cust_company = Sk.icustcompany_B2C
    @c_paymode = Sk.cpaymode_COD
    @c_processing = Sk.iinreturn_cprocessing_WAIT
    @c_redelivery = Sk.credelivery_RETURN
  end

  def teardown
    super
    FileUtils.rm(@file) if File.exist?(@file)
  end

  def run_batch(rc = OK)
    commit
    @batch = AmosWarehouse::Batches::CodReturnList.new('--disable-dprun-check -e 10')
    @batch.instance_variable_set(:@file_name, @file_name)
    assert_equal(rc, @batch.run)
  end

  # WIP: 
  isolate_testcase
  test 'success export file' do
    3.times do
      p_invsum = PInvsum.make(
        i_company: @i_company,
        i_cust_company: @i_cust_company,
        c_paymode: @c_paymode,
        i_parcel: Faker::Number::number(12),
        s_opcode: 'TST'
      )
      IInReturn.make(
        d_dp: @cdppara.d_dp,
        i_dpslot: @cdppara.i_dpslot,
        c_processing: @c_processing,
        c_redelivery: @c_redelivery,
        i_invoice: p_invsum.i_invoice
      )
    end

    run_batch
    assert_file
  end

  def assert_file
    assert(File.exist?(@file), "File #{@file} not found")
    content = File.open(@file, 'r').read.split("\n")
    assert_equal(@batch.instance_variable_get(:@record_count) + 1, content.size)
    content.each do |line|
      # assert_line(line)
    end
  end
end
```