---
layout: post
title: ActiveMailer PDF Email Attachments with Paperclip
date: 2015-08-08
---

Let's say we have a ```Report``` model, which has PDFs as Paperclip attachments:

``` ruby
# /app/models/report.rb
class Report < ActiveRecord::Base
  belongs_to :user
  has_attached_file :pdf
  validates_attachment :pdf, content_type: { content_type: "application/pdf" }
end
```

We want to automate sending out a report by email each week to the user who created the report (this post won't cover the automation; use [something to automate that](https://www.ruby-toolbox.com/categories/scheduling), like a CRON job, or the [Clockwork gem](https://github.com/tomykaira/clockwork)).

For this example, I've set up an ActionMailer mailer to send the email, and an ActiveJob job to generate the PDF from values in the Report object.

## Creating the ActiveJob job

``` ruby
rails g job generateReportPdf
```

``` ruby
# /app/jobs/generate_report_pdf_job.rb
class GenerateReportPdfJob < ActiveJob::Base
  queue_as :default

  def perform(args)
    @report = Report.find args[:report_id]
    # create an instance of ActionView, so we can use the render method outside of a controller
    av = ActionView::Base.new()
    av.view_paths = ActionController::Base.view_paths

    # need these in case your view constructs any links or references any helper methods.
    av.class_eval do
      include Rails.application.routes.url_helpers
      include ApplicationHelper
    end

    pdf_html = av.render template: 'reports/generate_pdf', layout: 'layouts/pdf', locals: {report: @report}

    # use wicked_pdf gem to create PDF from the doc HTML
    doc_pdf = WickedPdf.new.pdf_from_string(
      pdf_html,
      page_size: 'Letter',
      javascript_delay: 6000
    )

    # save PDF to disk
    pdf_path = Rails.root.join('tmp/reports', "#{@report.id}_#{Date.today.iso8601}.pdf")
    File.open(pdf_path, 'wb') do |file|
      file << doc_pdf
    end

    @report.pdf = File.open pdf_path
    @report.save!

    # The report has now been saved elsewhere using Paperclip; we don't need to store it locally
    File.delete(pdf_path) if File.exist?(pdf_path)
```

## Creating the ActionMailer mailer

Now we create the mailer which will call out to this ActiveJob job to generate the PDF:

``` ruby
rails g mailer reportPdf
```

``` ruby
# /app/mailers/report_pdf.rb
class ReportPdf < ApplicationMailer
  def send_mail(report_id:)
    @report = Report.find report_id

    GeneratePdfJob.perform_now(report_id: @report.id)
    @report.reload

    attachments["report_#{Date.today.iso8601}.pdf"] = {
      mime_type: 'application/pdf',
      content: open(@report.pdf.expiring_url).read
    }

    mail(
      to: @report.user.email,
      subject: "Weekly PDF Email: #{Date.today.iso8601}",
    )
  end
end
```

So now all we need to do is call ```ReportPdf.send_mail(report_id: @report.id)``` to generate and send an email with a PDF attachment, powered by Paperclip. The PDF generation is done via the [wicked_pdf gem](https://github.com/mileszs/wicked_pdf).